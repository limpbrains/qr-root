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

**Last synchronized commit**: Not yet synced
**Last sync date**: N/A
**Sync status**: Not yet synced

To perform the first sync, ask Claude to "sync qr" or use the sync-qr skill.

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
