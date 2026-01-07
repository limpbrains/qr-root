# QR Code Libraries Monorepo

A monorepo containing QR code-related projects as git submodules, with automated synchronization tooling.

## Overview

This repository aggregates three QR code-related projects:

- **[qr](https://github.com/limpbrains/qr)** - Pure Kotlin QR code reader library (decode-only)
- **[qr-paulmillr](https://github.com/paulmillr/qr)** - Original TypeScript QR encoder/decoder library by Paul Miller
- **[react-native-camera-kit-no-google](https://github.com/limpbrains/react-native-camera-kit-no-google)** - React Native camera library with QR scanning

## Quick Start

### Clone with Submodules

```bash
git clone --recurse-submodules git@github.com:limpbrains/qr-root.git
cd qr-root
```

If already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Update Submodules

```bash
# Update all submodules to latest
git submodule update --remote

# Update specific submodule
git submodule update --remote qr
```

## Submodules

### qr (Kotlin)

Pure Kotlin library for QR code decoding from raw pixel data.

**Features**:
- No external dependencies
- Supports Grayscale, RGB, and RGBA input
- Optimized for camera frames (direct Y-plane processing)
- All QR versions (1-40) and error correction levels (L, M, Q, H)
- Reed-Solomon error correction

**Build & Test**:
```bash
cd qr
./gradlew build
./gradlew test
```

**Documentation**: See [qr/CLAUDE.md](qr/CLAUDE.md) and [qr/README.md](qr/README.md)

### qr-paulmillr (TypeScript)

Original minimal 0-dependency QR code generator & reader by Paul Miller.

**Features**:
- Encoding: ASCII, term, GIF, SVG, PNG
- Decoding: Camera feed, files, non-browser environments
- Fast: Faster than all JS implementations
- Extensive test vectors (100MB+)

**Install & Build**:
```bash
cd qr-paulmillr
npm install
npm run build
npm test
```

**Documentation**: See [qr-paulmillr/README.md](qr-paulmillr/README.md)

### react-native-camera-kit-no-google

High-performance React Native camera library with barcode/QR scanning.

**Features**:
- Photo capture
- Barcode/QR code scanning
- Camera preview (including iOS simulator)
- Camera controls (flash, focus, zoom, torch)
- Cross-platform (iOS & Android)

**Build & Test**:
```bash
cd react-native-camera-kit-no-google
yarn install
yarn build
yarn test
yarn lint
```

**Documentation**: See [react-native-camera-kit-no-google/CLAUDE.md](react-native-camera-kit-no-google/CLAUDE.md)

## Automated Synchronization: `sync-qr` Skill

This repository includes a custom Claude Code skill for automatically synchronizing decode-related changes from the JavaScript library (qr-paulmillr) to the Kotlin port (qr).

### What is sync-qr?

The `sync-qr` skill is an intelligent code porting tool that:

1. **Detects changes** in qr-paulmillr since the last sync
2. **Filters decode-only changes** (ignores encoding, DOM, browser-specific code)
3. **Translates JavaScript to Kotlin** while preserving algorithmic correctness and Kotlin idioms
4. **Synchronizes tests** between Node.js test framework and JUnit 5
5. **Runs tests** and attempts automatic fixes for common issues
6. **Tracks sync state** in CLAUDE.md

### Usage

Using [Claude Code](https://claude.com/claude-code), simply ask:

```
sync qr
```

Or use any of these trigger phrases:
- "sync libraries"
- "port JS changes to Kotlin"
- "update Kotlin from JavaScript"

### Workflow

The skill follows a 10-step automated workflow:

1. **Check Sync State** - Read last synced commit from CLAUDE.md
2. **Pull Latest** - Update qr-paulmillr submodule
3. **Detect Changes** - Compare commits, identify relevant changes
4. **Analyze Changes** - Filter decode-related changes
5. **Port Changes** - Translate JS→Kotlin using intelligent patterns
6. **Sync Tests** - Update corresponding Kotlin tests
7. **Run Tests** - Execute `./gradlew test`
8. **Auto-Fix** - Attempt fixes for common test failures (up to 3 iterations)
9. **Update State** - Record sync results in CLAUDE.md
10. **Commit** - Optionally commit changes with detailed message

### Key Features

#### Intelligent Change Detection

**Includes**:
- `src/decode.ts` - All decoding logic
- `src/index.ts` - Shared utilities (Bitmap, Transform, GF, RS, etc.)
- Decode-related tests

**Excludes**:
- Encoding functions
- DOM/browser utilities
- Encoding-only tests

#### Smart Translation

The skill uses comprehensive translation patterns to convert JavaScript idioms to Kotlin:

- Type mappings (`Uint8Array` → `ByteArray`, `number` → `Int`/`Double`)
- Syntax conversions (`for` loops → ranges, bitwise ops → `and`/`or`/`xor`)
- Kotlin idioms (`when` expressions, null safety, operator overloading)
- Style preservation (object singletons, data classes, `val` over `var`)

#### Critical Safeguards

**Bitmap coordinate swap**: JavaScript uses `b.data[y][x]` but Kotlin uses `b.get(x, y)` - the skill automatically handles this swap.

**Assertion order**: JUnit has `assertEquals(expected, actual)` but JavaScript has `assert.strictEqual(actual, expected)` - parameters are swapped!

**Kotlin enhancements**: Preserves Kotlin-specific optimizations (grayscale input, triangle validation, threshold retry)

#### Auto-Fix Capabilities

Automatically fixes common test failures:

1. **Off-by-one errors** - `0..n` → `0 until n`
2. **Type mismatches** - Add `.toInt()`, `.toDouble()`
3. **Null safety** - Add `?`, `!!`, `?.`, `?:`
4. **Bitmap access** - Verify x/y order
5. **Encoding issues** - Port ECI support, charset handling

### Skill Documentation

Complete documentation is in `.claude/skills/sync-qr/`:

- **[SKILL.md](.claude/skills/sync-qr/SKILL.md)** - Complete workflow and instructions
- **[FILE_MAPPING.md](.claude/skills/sync-qr/FILE_MAPPING.md)** - JS→Kotlin file correspondence
- **[TRANSLATION_PATTERNS.md](.claude/skills/sync-qr/TRANSLATION_PATTERNS.md)** - Code translation patterns and style guide
- **[TEST_STRATEGY.md](.claude/skills/sync-qr/TEST_STRATEGY.md)** - Test synchronization guidelines

### Example: Syncing ECI Support

When qr-paulmillr added ECI (Extended Channel Interpretation) encoding support:

**JavaScript** (qr-paulmillr commit `0fbd0a4`):
```javascript
const eciToEncoding = { 1: 'iso-8859-1', 26: 'utf-8', 20: 'shift-jis' };
function decodeWithEci(bytes: Uint8Array, eci: number = 26): string {
  const encoding = eciToEncoding[eci];
  if (!encoding) throw new Error(`Unsupported ECI: ${eci}`);
  return new TextDecoder(encoding).decode(bytes);
}
```

**Kotlin** (automatically translated):
```kotlin
private val eciToEncoding = mapOf(
  1 to "iso-8859-1",
  26 to "utf-8",
  20 to "shift-jis"
)

private fun decodeWithEci(bytes: ByteArray, eci: Int = 26): String {
  val encoding = eciToEncoding[eci]
    ?: throw QRDecodingException("Unsupported ECI: $eci")
  return String(bytes, Charset.forName(encoding))
}
```

The skill:
- Identified the change in `src/decode.ts`
- Mapped it to `BitDecoder.kt`
- Translated TypeScript to Kotlin with proper idioms
- Updated corresponding tests
- Verified all tests pass

### Sync State Tracking

Sync state is tracked in [CLAUDE.md](CLAUDE.md#qr-sync-state):

```markdown
## QR Sync State

Last synchronized commit: 0fbd0a4
Last sync date: 2026-01-07T10:30:00Z
Sync status: success
Changes ported:
- Added ECI encoding support
- Added inverted QR support
- Performance optimizations

Test results:
- VectorTest: 9134/9281 (98.42%)
- ImageDecodingTest: 113/118 (95.76%)
```

## Contributing

### Adding Submodules

```bash
git submodule add -b <branch> <repository-url> <path>
git submodule add -b main git@github.com:limpbrains/qr.git qr
```

### Updating Sync State

After manually porting changes, update the sync state in CLAUDE.md:

```markdown
## QR Sync State

Last synchronized commit: <commit-hash>
Last sync date: <ISO-8601-date>
Sync status: success
Changes ported:
- Brief description of changes
```

## License

This monorepo itself is MIT licensed. Each submodule has its own license:

- **qr**: Dual-licensed Apache 2.0 OR MIT
- **qr-paulmillr**: Dual-licensed Apache 2.0 OR MIT
- **react-native-camera-kit-no-google**: MIT

See individual submodule LICENSE files for details.

## Acknowledgments

- **Paul Miller** ([@paulmillr](https://github.com/paulmillr)) - Original QR library author
- **ZXing** - Algorithm inspiration and reference implementation
- **BoofCV** - QR code test vectors

## Links

- [Claude Code](https://claude.com/claude-code) - AI-powered coding assistant
- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [QR Code Specification (ISO/IEC 18004)](https://www.iso.org/standard/62021.html)
- [QR Code Test Vectors](https://github.com/paulmillr/qr-code-vectors)
