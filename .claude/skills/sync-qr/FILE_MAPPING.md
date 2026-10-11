# JavaScript → Kotlin File Mapping

Correspondence between qr-paulmillr (TypeScript) and the Kotlin port in `qr/`, as of the **v0.7.2**
port (`951806d`). Line numbers are for v0.7.2 `src/decode.ts` / `src/index.ts`; every Kotlin
function carries a `decode.ts Lxxx-yyy` comment naming its source, so after an upstream change
`grep -rn "decode.ts L" qr/src/main` finds the Kotlin side of any changed JS range.

## Source: `src/decode.ts` (3457 lines)

| JS (lines) | Kotlin file | Kotlin symbol |
|---|---|---|
| `FAIL` (L193-202) | `Types.kt` | `enum class Fail` |
| `ECI_ENCODINGS`, `*_LENGTH_BITS`, `Payload.create/decode/finish` (L206-369) | `Payload.kt` | `class Payload` (`decode`, `clean`, `ECI_CHARSETS`) |
| `mul`/`inv` (L130-131) | `GF256.kt` | `GF256.mul`, `GF256.inv` |
| `dist2`, `distance` (L372-375) | `Homography.kt` | `dist2`, `distance` (+ V8 `hypot`) |
| `checkVersion` (L376-393) | `GridDecoder.kt` | `GridDecoder.checkVersion` |
| `squareToQuad`, `adjugate`, `ptMul` (L398-462), `packQuad` (L698-718), `mapPoint` (L905), `snapSize` (L1273) | `Homography.kt` | same names (+ V8 `cbrt`, `jsRound`) |
| `validateSize`, `validateImage`, `copyWords`/`copyTriples`/`copyLuma` (L142-153, L553-695) | `QRScanner.kt` | `validateSize`, `copyLuma` (plain loops) |
| `bit` (L719), `growFinders` (L938-951) | `Layer.kt` | `Layer.bit`, `growFinders` |
| `ratio`, `run`, `cross`, `vertical`, `refinePattern`, `refineTriple`, `fitPattern` (L724-902) | `Finder.kt` | same names |
| `crossPitch`, `finderPitch`, `edgePitch` (L913-936), `copyPattern` (L953-958) | `Finder.kt` | same names |
| `swapSet`, `siftDown` (L960-982) | `Selection.kt` | `swapSet`, `siftDown` |
| `evalLow` (L984-988) | `RS.kt` | `RS.evalLow` |
| `scanRows.blocks` / `scanRows.bitmap` (L1036-1182) | `Layer.kt` | `computeBlocks`, `computeBitmap` (+ `resizeFrom` for the pyramid) |
| `scanRows.find` (L1184-1269) | `Finder.kt` | `Finder.find` |
| `runDecode`/`runDecodeAsync`/`nextTick` (L1276-1313) | — | not ported (sync only) |
| `_QRScanner` constructor, `addImage`, `stage`, `clean` (L1315-1632) | `QRScanner.kt` | `QRScanner` init, `addImage`, `processLuma`, `stage`, `clean` |
| `mapQuad`, `mapFinderQuad`, `reserve`, `setAlignments` (L1474-1517) | `Projection.kt` | same names (extension funs on `QRScanner`) |
| `read`, `perspective`, `searchAlign` (L1635-1712) | `Projection.kt` | same names |
| `excluded`, `exclude`, `retain`, `makeTriple`, `pickPolarity` (L1715-2014) | `Selection.kt` | same names |
| `timing`, `findBasicAlign` (L2017-2081) | `Projection.kt` | same names |
| `report` (L2084-2153) | — | not ported (`pointsOnDetect`) |
| `upgrade` (L2155-2175) | `Projection.kt` | `upgrade` |
| `decodeGrid`, `decodeOriented` (L2182-2483) | `GridDecoder.kt` | `decodeGrid`, `decodeOriented`, `decodeFormat` |
| RS corrector inside `decodeOriented` (L2369-2468) | `RS.kt` | `RS.correct` |
| `projectQuad`, `projectMap`, `confirm`, `projectWith` (L2487-2793) | `Projection.kt` | same names (+ `project`, `tiles` for v7+ tiles) |
| `scan` round 0 (L2796-2893) | `QRScanner.kt` / `Selection.kt` | `QRScanner.scan`, `pickMandatory` |
| `scan` `schedule:` block + nextSet loop (L2895-3024) | `Selection.kt` | `buildSchedule`, `nextSet` |
| `scan` success bookkeeping / exclusion (~L3025-3098) | `QRScanner.kt` | `QRScanner.scan` |
| `walk`, `decode` (L3100-3133) | `QRScanner.kt` | `QRScanner.decode(all)` |
| `decodeQR` (L3135-3150) | `QRDecoder.kt` | `QRDecoder.decode` |
| `decodeQRBatch` (L3152-3190) | — | not ported (async); covered by reusing one `QRScanner` |
| `BarcodeDetector` & canvas helpers (L3192-end) | — | not ported (browser) |
| `DecodeOpts` validation (L522-551) | `QRScanner.kt` | constructor `require`s |

## Source: `src/index.ts` (spec tables and RS)

| JS (lines) | Kotlin file | Kotlin symbol |
|---|---|---|
| `info.BYTES` closed form (L164-176) | `QRInfo.kt` | `QRInfo.BYTES` (table) |
| `ECC_LEVELS` (L178), `WORDS_PER_BLOCK` (L181-198), `ECC_BLOCKS` (L201-206) | `QRInfo.kt` / `Types.kt` | `WORDS_PER_BLOCK`, `ECC_BLOCKS` indexed by `ErrorCorrection.ordinal` |
| `alphabet` alphanumeric (L212) | `QRInfo.kt` | `QRInfo.ALPHANUMERIC` |
| `formatBits`, `versionBits` | `QRInfo.kt` | same names |
| `alignmentPatterns`, `size.encode` | `src/test/.../TestEncoder.kt` | test-only `TestEncoder.alignmentPatterns`, `sizeEncode`; the decoder uses `Projection.setAlignments`, a direct port of decode.ts L1505-1517 |
| `EXP`/`LOG` (L263-273) | `GF256.kt` | `GF256.EXP` (doubled, 510), `GF256.LOG` |
| `rsGenerator`, `RS_CACHE` (L276-312) | `RS.kt` | `RS.cached` (`rsGenerator` is private in RS.kt) |
| `rsEcc` (L318-336), `encodeData`, `encodeQR`, `penalty` | `src/test/.../TestEncoder.kt` | test-only encoder |
| mask predicates `maskCalc` (L446-472) | `QRInfo.kt` | `maskBits` (72-entry table) |
| everything else (encoding, SVG/GIF output) | — | not ported |

`src/dom.ts` (browser camera/canvas) is not ported.

## Tests

| JS test | Kotlin test(s) |
|---|---|
| `test/decode.test.ts` (DECODED table, image sweeps) | `ImageDecodingTest` (+ generated `ImageVectors.kt`), `RealImageTest` |
| `test/decode.test.ts` (synthetic symbols, options, errors) | `SyntheticTest`, `QRScannerTest`, `QRDecoderTest` |
| `test/decode-batch.test.ts` | `QRScannerTest` (scanner reuse, all mode, batch order) |
| `test/utils.test.ts` | `QRInfoTest`, `RSTest`, `PayloadTest` |
| `test/misc/jpg.ts`, `test/misc/png.ts` | `JpegLuma.kt`, `PngLuma.kt` |
| `test/jpg.test.ts`, `test/png.test.ts` | `LumaParityTest` (luma CRC guard) |
| `test/decode-async.test.ts`, `dom*.test.ts`, `encode.test.ts`, `gif.test.ts`, `polyfill.test.ts` | not ported |
| `encodeQR` fixtures | `TestEncoder.kt`, `TestEncoderTest` |
| small vectors (qr-code-vectors) | `VectorTest` |
| JS stage dumps (`tools/js-dump`) | `LayerDumpTest`, `FinderDumpTest`, `ScheduleDumpTest`, `HomographyTest` |

## Kotlin-only

| Feature | Location |
|---|---|
| Gray / Y-plane input with offset + stride, `processLuma` zero-copy | `QRScanner.addImage`, `copyLuma`, `Image.format` |
| Synchronous reusable `QRScanner` API (`decode(all)`, `lastFailure`) | `QRScanner.kt` |
| Exception facade over `Fail` tags, growing shared scanner | `QRDecoder.kt` |
| ECI 2 via Java `IBM437` (JS fails: no TextDecoder label) | `Payload.ECI_CHARSETS` |

The old Kotlin-only heuristics (threshold-offset retry, triangle validation, relaxed finder variance)
were dropped with the v0.7.2 port; do not reintroduce them.
