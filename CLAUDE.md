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

**Last synchronized commit**: d846c9d2f5d9f2ff82b775c8a1b061fed37abea0
**Last sync date**: 2026-04-20
**Sync status**: success

**Changes ported** (from commit 0fbd0a4..d846c9d, 11 commits scanned, 1 ported):
- `dd46f5b` Speed-up decoding 4x — ported the correctness-affecting parts of the `toBitmap()` rewrite:
  - Per-block min/max/average tracked together as packed Int state
  - Color-spread sampling (`spreadMean`) from RGB data as a scene-type signal
  - Scene gates: `hiRangeFrac`/`veryLowFrac`, derived `spotBias` and `useVarField`
  - Summed-area tables for 5x5 neighborhood mean (`integ`) and variance (`integSqr`)
  - Per-block cut = `average + localAdj + chromaAdj + varAdj + spotBias + scene-nudges`, clamped to `[min, max]`
  - Fast paths: `average < min` → block stays light; `average >= max` → block is fully dark
- Skipped pure-performance JS-only tricks that depend on qr-paulmillr's Uint32Array Bitmap layout: SUM16/MIN16/MAX16 LUTs, Uint32Array aliasing over brightness/RGBA, packed Bitmap writes, `ctz32`-based scanLine run-skipping. Kotlin `Bitmap` is `Array<Array<Boolean?>>`, so the LUT/word tricks have no equivalent without a full Bitmap rewrite.
- Other 10 commits skipped: TypeScript version bumps (d846c9d, 2b1ef95), release bumps (d137596, a3abe54), JSDoc-only (5e1a5d3), `src/dom.ts` `gifToPng` browser feature (780a672, 11bf143), README (513cfad, 898591d), bun-only ECI test skip (11ef042).

**Kotlin-only features preserved**: grayscale (1 bpp) direct Y-plane input, `thresholdOffset` retry parameter, triangle validation, relaxed finder variance retries.

**Test results** (2026-04-20):
- Build: SUCCESS ✓
- VectorTest: 9134 / 9281 passed (98.42%) ✓ Baseline maintained
- Batch decode test: 50/50 passed ✓
- All unit tests (BitmapTest, GaloisFieldTest, ReedSolomonTest, QRInfoTest, QRDecoderTest): PASSED ✓
- Failing VectorTest cases are identical to pre-sync baseline (same known-bad test vectors; both JS and Kotlin decoders fail on them — documented in qr/CLAUDE.md).

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
