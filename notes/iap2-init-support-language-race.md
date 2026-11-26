# The `com.tcc.iap2server` disconnect crash: a race on the language statics

Notes on a stock bug in the CarPlay/iAP2 app on Gen5W units, found from drive logs
(2026-11). Reported as issue #8. Written up here because the fix touches smali we do
not carry sources for, and because the reasoning went through three rounds of
adversarial review and had to be revised twice. Read the caveats, not just the fix.

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
back. Which is what you would expect — the app dies during teardown, Android restarts
it, and the fresh process has clean statics. The "workaround" is really completing a
teardown the app cannot finish itself.

## The mechanism

`InitSupportLanguage()` builds a two-element array, stores it, and then **re-reads the
static field before each write** instead of keeping the array it just made:

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

A second method sizes the same field from a Parcel count, so a count of 0 installs a
zero-length array:

```smali
0x0158 invoke-virtual     v14, Landroid/os/Parcel;->readInt()I
0x015e sput               v0, ->iAP2LanguageListCnt I
0x0164 new-array          v0, v0, [Ljava/lang/String;      # String[count]
0x0168 sput-object        v0, ->iAP2LanguageName [Ljava/lang/String;
```

Only two writers of those two fields exist in all 1246 methods, and `readFromParcel`
is the only code that can install a zero-length array. `index=1` pins the throw to the
`aput-object` at file `0x03b162`.

Threading is real: `InitSupportLanguage` runs on `Iap2Service$MsgConsumer` (a
`Thread`); `UpdateCurrentLanguage` is reached from an AIDL `IIap2Service$Stub`, i.e. a
Binder thread. The class has **zero monitors**, and there is an allocation and a
`Log.d` between the guard and the index, so the window is wide.

## Caveat on the mechanism — read this

**`readFromParcel` has no caller in either build we hold.** `Parcelable.CREATOR`
returns the singleton `mInfoMsg` without unmarshalling, and neither `<init>(Parcel)`
nor `msgInfoReadFromParcel` is reachable, by direct call or by reflective dispatch.
The crash proves that writer ran, so **the crashing binary is not one of the two we
have**, and the path that reaches the writer in the shipped build is unproven.

Two knock-on effects:

- The trace's line numbers cannot be matched against our dexes; their line tables do
  not resolve, which is consistent with the crashing build differing.
- The writer is itself racy: `readFromParcel` re-reads the statics it just wrote at
  `0x0168`, `0x0176` and `0x017e`, so a concurrent swap can desynchronise the Parcel
  mid-parse.

The method is pinned by the frame name and both held builds carry a byte-identical
instruction sequence for it, so the site is right. But the argument is not closed, and
it should not be described as closed.

## The fix

### Minimal — the reported site (3 substitutions)

Each 4 bytes in and 4 bytes out, so no offsets move:

```
0x0008  sget v0, ->iAP2LanguageListCnt I     -> const/16 v0, 2
0x0014  sget-object v0, ->iAP2LanguageName   -> move-object/from16 v0, v0
0x0020  sget-object v0, ->iAP2LanguageName   -> move-object/from16 v0, v0
```

`move-object` rather than `move` is what keeps the reference type for the verifier, and
a self-move is legal. Afterwards the method reads no mutable statics at all.

`a4974bf76ad77c4954815ec94704ec5e` -> `b5b0850b5e590363b2dd8996f3606c22` (bridge);
`c20fccd22cb102f5ff36c0362f02f81b` -> `5b1c3c5ddbd9c31882dc1353db39d30b` (+ev).

### Extended — closes the class (26 edits over 4 `code_item`s)

| method | edits | cost |
|---|---|---|
| `InitSupportLanguage` | 3 | same-size, no header change |
| `getSupportLanguage` | 3 | same-size; keep the guarded array in the dead `v5` |
| `readFromParcel` | 6 | same-size |
| `UpdateCurrentLanguage` | 14 | needs a 9th register: `registers_size` 8 -> 9 in the `code_item` header plus 9 renumberings |

Dex length unchanged and no instruction offset moves anywhere (checked across all 1246
and 1303 methods). The first three are low risk; the fourth changes a method header and
renumbers registers, so it wants an ART verifier pass and an on-car run. Recommend
taking the three-method set, or all four only with that verification.

Synchronising instead is the right long-term shape but not deliverable from these
artefacts: it needs a new static field (table change, full rebuild), `monitor-enter`
with catch-all handlers in four methods that have `tries_size=0`, and lock ordering
against `mInfoMsg`, shared by 7 classes whose external callers are not in this APK.

## What a patch file is worth here

Applying a `.lpd` is a delta from the car's own stock file. The source is md5- and
length-checked before the payload is decoded; a control block of `(x=0, y=len, z=0)`
with an empty diff means the delta does not *read* the source, so a replacement can be
built without holding the OEM file, provided the header's stock md5/length are right.

Three reasons not to treat that as the deliverable:

1. **22 `.lpd` files** are involved and we hold 2. The source md5 is per car.
2. **`tools/mkpatches.py` has no `write_nosrc`** — it always emits bsdiff deltas, so a
   hand-made no-source patch is replaced the next time patch sets are regenerated.
3. **CI cannot validate patch content.** `tools/lpatch/test.sh` only rebuilds the real
   EV6 apps when stock files are present at `STOCK` (default `~/hmg-private/stock/ev6`).
   The manifest and `compat.json` bookkeeping is checked; the patch itself is not.

The durable artefact is the **edit**, regenerated by the maintainer from his own stock
files. A correct change also touches 11 manifests' field 5, `patches/universal/manifest`
field 6, and 10 variant lists in `docs/data/compat.json`.

## Verification done

- The three substitutions were re-derived independently with a hand-written DEX parser
  and reproduce both fixed dexes byte-for-byte; the patched method reads no mutable
  static and the header's `checksum` (adler32 of bytes[12:]) and `signature` (SHA-1 of
  bytes[32:]) are correct, in that order.
- The captured APKs are byte-exactly `zipadd`'s output, so replacing the stored
  `classes.dex` and its two CRC fields yields the fixed result APK: `b8d6001c…`
  (bridge) and `78f9c47a…` (+ev).
- The stock APK was reconstructed exactly (26 624 B, `fde20843…`, matching
  `patches/ev6/stock`), which is how the `zipadd` constants could be confirmed.
- Byte-diff shows only the three code regions and the 24-byte header differ.

Not done: any ART verifier run, any on-car run, and `tools/mock/install-test.sh`.

## Dead end, recorded so it is not repeated

An earlier version of this work flagged `ArHudTrap.originalInts` as a sibling case —
two methods writing one static int array. It is **not** a race and was retracted:
both writers are `public static synchronized` *and* `monitor-enter` the class monitor,
the only reader is called solely from those methods and `tick`, and the writers have no
invoke site at all in DiPO or GPM. The shape is not the bug; the missing lock is.
Note also that `synchronized` on a method is an **access flag**, not a `monitor-enter`
instruction, so counting monitor ops does not tell you whether a method is synchronised.
