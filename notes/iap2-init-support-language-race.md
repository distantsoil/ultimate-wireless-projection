# The `com.tcc.iap2server` disconnect crash: a race on the language statics

Notes on a stock bug in the CarPlay/iAP2 app on Gen5W units, found from drive logs
(2026-11). Reported as issue #8. Written up here because the fix touches smali we
do not carry sources for, and the reasoning is worth keeping.

## The crash

Android dropbox, `system_app_crash@1795212242449` — **2026-11-20 22:04:02Z** — and a
second record, `@1795369173789`, **2026-11-22 17:39:33Z**, byte-identical body. Two
separate disconnects, two days apart:

```
Process: com.tcc.iap2server
java.lang.ArrayIndexOutOfBoundsException: length=0; index=1
	at com.tcc.iap2server.Iap2InitInformationMsg.InitSupportLanguage(Iap2InitInformationMsg.java:1179)
	at com.tcc.iap2server.Iap2InitInformationMsg.initVariable(Iap2InitInformationMsg.java:585)
	at com.tcc.iap2server.Iap2InitInformationMsg.init(Iap2InitInformationMsg.java:535)
	at com.tcc.iap2server.Iap2Service.setComponent(Iap2Service.java:1876)
	at com.tcc.iap2server.Iap2Service.initVariableForDisconnect(Iap2Service.java:1021)
	at com.tcc.iap2server.Iap2ConnectionManager$Iap2Connection.finish(Iap2ConnectionManager.java:255)
	at com.tcc.iap2server.Iap2ConnectionManager.requestIap2Disconnect(Iap2ConnectionManager.java:179)
	at com.tcc.iap2server.Iap2Service.onFinishMain(Iap2Service.java:1834)
	at com.tcc.iap2server.Iap2Service$MsgConsumer.run(Iap2Service.java:2200)
```

Symptom on the car: wireless CarPlay dies, and a forced reconnect always brings it
back. Which is what you would expect — the app dies during teardown, Android
restarts it, and the fresh process has clean statics. The "workaround" is really
completing a teardown the app cannot finish itself.

## Why it happens

`InitSupportLanguage()` builds a two-element array, stores it, and then **re-reads
the static field before each write** instead of keeping the array it just made:

```smali
0x0000 const/4            v3, 0
0x0002 const/4            v0, 2
0x0004 sput               v0, ->iAP2LanguageListCnt I
0x0008 sget               v0, ->iAP2LanguageListCnt I
0x000c new-array          v0, v0, [Ljava/lang/String;      # length 2
0x0010 sput-object        v0, ->iAP2LanguageName [Ljava/lang/String;
0x0014 sget-object        v0, ->iAP2LanguageName           # read A
0x0018 const-string       v1, "en"
0x001c aput-object        v1, v0, v3                       # A[0]
0x0020 sget-object        v0, ->iAP2LanguageName           # read B  <-- can differ
0x0024 const/4            v1, 1
0x0026 const-string       v2, "ko"
0x002a aput-object        v2, v0, v1                       # B[1]  <-- throws if B is length 0
0x002e sput               v3, ->iAP2CurrentLanguage I
0x0032 return-void
```

A second method writes the same statics from a Parcel, and sizes the array from
whatever the Parcel said:

```smali
0x0158 invoke-virtual     v14, Landroid/os/Parcel;->readInt()I
0x015e sput               v0, ->iAP2LanguageListCnt I
0x0164 new-array          v0, v0, [Ljava/lang/String;      # String[count]
0x0168 sput-object        v0, ->iAP2LanguageName [Ljava/lang/String;   # 0-length if count is 0
```

An exhaustive scan of all 1246 methods finds **only these two writers** of those two
fields anywhere in the dex, and `readFromParcel` is the only code that can install a
zero-length array. `index=1` pins the throw to the `aput-object` at file `0x03b162`.

Threading: the crash path is `Iap2Service$MsgConsumer.run`, while `readFromParcel` is
reachable only through `Parcelable$Creator.createFromParcel` (framework
unmarshalling). `Iap2InitInformationMsg` contains **zero** monitor operations. So one
thread re-initialises while the Binder thread replaces the field, and the two
`sget-object`s in between straddle the replacement.

## The fix applied

Three substitutions, each 4 bytes in and 4 bytes out, so no offsets move:

```
0x0008  sget v0, ->iAP2LanguageListCnt I     -> const/16 v0, 2
0x0014  sget-object v0, ->iAP2LanguageName   -> move-object/from16 v0, v0
0x0020  sget-object v0, ->iAP2LanguageName   -> move-object/from16 v0, v0
```

`move-object` rather than `move` is what keeps the reference type for the verifier,
and a self-move is legal. Afterwards the method reads no mutable statics at all: it
builds its own array and writes into that.

Result: `classes.dex` `a4974bf76ad77c4954815ec94704ec5e` ->
`b5b0850b5e590363b2dd8996f3606c22`, same length 311084. For the `+ev` build,
`c20fccd22cb102f5ff36c0362f02f81b` -> `5b1c3c5ddbd9c31882dc1353db39d30b`.

Applying a `.lpd` is a delta from the car's own stock file, and the source is only
ever md5-checked — a patch whose control block is `(x=0, y=len, z=0)` carries the
whole result in the extra block and never reads the source. So the replacement
patches can be produced without holding the OEM file, as long as the header's stock
md5/length are right. The result dex is also the **same on every car** for a given
kind (`a4974bf7…` across `ev6`, `neev`, `neev_ca`, `nq5`, `lx2pe`, `cvev_kr`,
`neev_kr`, `dl3hev`, `universal`), so one fixed dex serves all of them.

## What this does not fix

The other readers of the same unsynchronised statics have the identical shape —
`array-length` + guard on one read, then a **second** read that is indexed. So the
same race can still throw there:

- `getSupportLanguage` — `sget-object` at insn 1 (guarded with `array-length`), then
  `sget-object` at insn 9 immediately followed by `aget-object`.
- `UpdateCurrentLanguage` — guarded read at insn 20, then `sget-object` at insns 31
  and 38, both immediately followed by `aget-object`.

So the three substitutions close the site in the trace, not the class. A complete fix
would stop the statics being replaced under a reader — either by not re-reading after
the guard in those two methods as well, or by synchronising the writers and readers.
Synchronisation is the correct shape but is a much larger smali change and risks lock
ordering with unknown callers, so it is a maintainer decision.

## Verification done

- The patched dex reparses and disassembles as expected; dex header `checksum`
  (adler32 of bytes[12:]) and `signature` (SHA-1 of bytes[32:]) recomputed.
- Byte-diff against the input shows exactly three 4-byte code regions plus the
  24-byte header region differ.
- The replacement `.lpd` was built with `tools/mkdelta.py` (which asserts its own
  round trip) and applying it with a deliberately wrong source of the right length
  still yields the fixed dex, confirming the source content is unused.
- The rebuilt result APKs open as valid zips with the fixed dex inside.

Not done: `tools/mock/install-test.sh`. No Linux here, and it needs `qemu-arm-static`
and `smali`/`baksmali` — CI or a maintainer machine should be the judge.

## Two open questions

1. **The trace's line numbers cannot be matched against the dexes held here.** The
   captured builds' line tables do not resolve, so `:1179` cannot have come from
   them; the crashing build had intact debug info. The *method* is pinned by the
   frame name and both captured builds carry a byte-identical instruction sequence
   for it, so the diagnosis stands, but the exact binary is not in hand.
2. **The stock `.odex` embeds the same code and carries the dex SHA-1**, so replacing
   only `classes.dex` in the APK leaves any prebuilt odex stale. The install path
   already rebuilds the APK and lets the unit re-optimise, but it is worth a look
   before shipping.
