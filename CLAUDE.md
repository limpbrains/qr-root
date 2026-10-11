# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a monorepo containing QR code-related projects as git submodules:

| Submodule | Description | Language |
|-----------|-------------|----------|
| `qr` | Pure Kotlin QR code reader library | Kotlin |
| `qr-paulmillr` | Original QR encoder/decoder library | TypeScript |
| `react-native-camera-kit-no-google` | React Native camera with QR scanning | TypeScript/Swift/Kotlin |

## Submodule Commands

```bash
# Initialize all submodules after cloning
git submodule update --init --recursive

# Update all submodules to latest
git submodule update --remote
```

## Submodule Details

Each submodule has its own CLAUDE.md or README with detailed build and development instructions.

### qr (Kotlin)
- Build: `./gradlew build`
- Test: `./gradlew test`
- Tracks `main` branch of limpbrains/qr

### qr-paulmillr (TypeScript)
- Install: `npm install`
- Build/test via package.json scripts
- Original library from paulmillr/qr

### react-native-camera-kit-no-google
- Build: `yarn build`
- Test: `yarn test`
- Lint: `yarn lint`
- Has its own CLAUDE.md with detailed architecture docs

## QR Sync State

This section tracks the synchronization state between the JavaScript QR library (qr-paulmillr) and the Kotlin port (qr).

**Last synchronized commit**: 951806df842fb7c9214012a73b00e60b259243d3 (v0.7.2)
**Last sync date**: 2026-10-11
**Sync status**: success

**Changes ported** (d846c9d..951806d, qr v0.7.2): upstream rewrote the decoder (`src/decode.ts`,
~3450 lines) around a reusable `_QRScanner`; the Kotlin decoder was re-ported from scratch on
branch `decoder-v0.7.2` (qr 0.7.2) and the old pipeline (Bitmap/PatternDetector/Transform/
BitDecoder/ReedSolomon/Interleave) was deleted. New architecture, file by file:
- `QRScanner.kt` (luma arena, pyramid of up to 4 layers, scan loop, `decode(all)`), `QRDecoder.kt` facade
- `Layer.kt` (8x8 block min/max, cuts, packed bitmap), `Finder.kt` (both-polarity finder scan/refine),
  `Selection.kt` (mandatory triple, retry schedule, multi-QR exclusion)
- `Projection.kt` (dimension candidates, alignment, timing gate, fine upgrade, v7+ tiles),
  `Homography.kt` (V8 hypot/cbrt ports), `GridDecoder.kt` (format candidates, mirror retry),
  `RS.kt` + `GF256.kt` (Berlekamp-Massey), `Payload.kt` (segments, ECI), `QRInfo.kt` (spec tables)
- Parity tooling in `qr/tools` (JS dump scripts) and fixtures in `qr/src/test/resources`.

**Kotlin-only features now**: grayscale / camera Y-plane input with offset + row stride
(`addImage(w, h, data, GRAY, offset, stride)`, zero-copy `luma` + `processLuma`); the synchronous
`QRScanner` API (`decode(all)`, `lastFailure`) and the exception-mapping `QRDecoder` facade; ECI 2
decodes via IBM437 (JS fails).
**Dropped Kotlin-only features**: `thresholdOffset` retry, triangle validation, relaxed finder
variance retries (the v0.7.2 algorithm supersedes them).

**Test results** (2026-10-11, `./gradlew test`: 748 tests, 0 failures, ~18 s):
- Small vectors: 9281/9281 (previous baseline 9134)
- BoofCV/issues/misc, 565 images: default mode 565/565 and all mode 565/565 identical to the JS dumps
- JS DECODED table (536 detection images): default 311 = JS 311, all mode 326 = JS 326
- Luma CRC guard 539/539; stage dumps (layers, finders, schedule) bit-exact

### Sync Process

The sync-qr skill (located in `.claude/skills/sync-qr/`) automatically:
1. Detects changes in qr-paulmillr since last sync
2. Filters decode-related changes (ignores encoding)
3. Ports JavaScript code to Kotlin preserving style
4. Synchronizes test cases
5. Runs tests and attempts auto-fixes
6. Updates this section with results

### Reference Documentation

- **SKILL.md** - Complete sync workflow
- **FILE_MAPPING.md** - JS→Kotlin file correspondence
- **TRANSLATION_PATTERNS.md** - Code translation patterns
- **TEST_STRATEGY.md** - Test synchronization guidelines
