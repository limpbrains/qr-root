# JavaScript → Kotlin Translation Patterns

This document provides comprehensive patterns for translating JavaScript/TypeScript code to Kotlin while preserving algorithmic correctness and maintaining Kotlin coding style.

## Type System

| JavaScript | Kotlin | Notes |
|------------|--------|-------|
| `number` | `Int` | For integers, array indices, counts |
| `number` | `Double` | For floating-point calculations, coordinates |
| `boolean` | `Boolean` | Same concept |
| `string` | `String` | Same concept |
| `Uint8Array` | `ByteArray` | 8-bit unsigned → signed bytes |
| `Uint32Array` | `IntArray` | 32-bit unsigned → signed ints |
| `number[]` | `IntArray` or `DoubleArray` | Use typed arrays for performance |
| `T[]` | `Array<T>` or `List<T>` | Generic arrays |
| `{ x: number, y: number }` | `data class Point(val x: Double, val y: Double)` | Use data classes |
| `type Union = A \| B` | `sealed class` or nullable | Context-dependent |
| `undefined` | `null` | Kotlin has no undefined |
| `T \| undefined` | `T?` | Nullable type |

## Function Definitions

| JavaScript | Kotlin |
|------------|--------|
| `function foo(x: number): number { return x * 2; }` | `fun foo(x: Int): Int = x * 2` |
| `const foo = (x: number) => x * 2` | `val foo = { x: Int -> x * 2 }` or `fun foo(x: Int) = x * 2` |
| `function foo(x?: number) { ... }` | `fun foo(x: Int? = null) { ... }` |
| `function foo(x: number = 10) { ... }` | `fun foo(x: Int = 10) { ... }` |
| `function foo(...args: number[]) { ... }` | `fun foo(vararg args: Int) { ... }` |

## Control Flow

| JavaScript | Kotlin |
|------------|--------|
| `if (cond) { ... } else { ... }` | `if (cond) { ... } else { ... }` |
| `const x = cond ? a : b` | `val x = if (cond) a else b` |
| `switch (x) { case 1: ...; break; }` | `when (x) { 1 -> ... }` |
| `for (let i = 0; i < n; i++)` | `for (i in 0 until n)` |
| `for (let i = n; i >= 0; i--)` | `for (i in n downTo 0)` |
| `for (let i = 0; i < n; i += 2)` | `for (i in 0 until n step 2)` |
| `while (cond) { ... }` | `while (cond) { ... }` |
| `do { ... } while (cond)` | `do { ... } while (cond)` |

## Arrays and Collections

| JavaScript | Kotlin |
|------------|--------|
| `new Uint8Array(n)` | `ByteArray(n)` |
| `new Array(n).fill(0)` | `IntArray(n)` or `Array(n) { 0 }` |
| `arr[i]` | `arr[i]` |
| `arr.length` | `arr.size` |
| `arr.push(x)` | `arr.add(x)` (MutableList only) |
| `arr.pop()` | `arr.removeAt(arr.size - 1)` |
| `arr.slice(start, end)` | `arr.sliceArray(start until end)` or `arr.slice(start until end)` |
| `arr.concat(other)` | `arr + other` |
| `arr.forEach(x => ...)` | `arr.forEach { x -> ... }` |
| `arr.map(x => ...)` | `arr.map { x -> ... }` |
| `arr.filter(x => ...)` | `arr.filter { x -> ... }` |
| `arr.reduce((acc, x) => ..., init)` | `arr.fold(init) { acc, x -> ... }` |
| `arr.find(x => ...)` | `arr.find { x -> ... }` |
| `arr.some(x => ...)` | `arr.any { x -> ... }` |
| `arr.every(x => ...)` | `arr.all { x -> ... }` |

## Bitwise Operations

| JavaScript | Kotlin |
|------------|--------|
| `a & b` | `a and b` |
| `a \| b` | `a or b` |
| `a ^ b` | `a xor b` |
| `~a` | `a.inv()` |
| `a << b` | `a shl b` |
| `a >> b` | `a shr b` (signed) |
| `a >>> b` | `a ushr b` (unsigned) |
| `n >>> 0` (convert to unsigned) | `n and 0xFFFF_FFFF` or `n.toUInt()` |
| `(a & 0xFF)` (mask to byte) | `a and 0xFF` |

## Math Operations

| JavaScript | Kotlin |
|------------|--------|
| `Math.sqrt(x)` | `sqrt(x)` (import kotlin.math.sqrt) |
| `Math.floor(x)` | `floor(x)` (import kotlin.math.floor) |
| `Math.ceil(x)` | `ceil(x)` (import kotlin.math.ceil) |
| `Math.round(x)` | `round(x)` (import kotlin.math.round) |
| `Math.min(a, b)` | `min(a, b)` (import kotlin.math.min) |
| `Math.max(a, b)` | `max(a, b)` (import kotlin.math.max) |
| `Math.abs(x)` | `abs(x)` (import kotlin.math.abs) |
| `Math.pow(x, y)` | `x.pow(y)` (import kotlin.math.pow) |

## String Operations

| JavaScript | Kotlin |
|------------|--------|
| `str.length` | `str.length` |
| `str.charAt(i)` | `str[i]` |
| `str.substring(start, end)` | `str.substring(start, end)` |
| `str.substring(start)` | `str.substring(start)` |
| `str.slice(start, end)` | `str.substring(start, end)` |
| `str.split(sep)` | `str.split(sep)` |
| `str.indexOf(sub)` | `str.indexOf(sub)` |
| `str.startsWith(prefix)` | `str.startsWith(prefix)` |
| `str.endsWith(suffix)` | `str.endsWith(suffix)` |
| `str.toUpperCase()` | `str.uppercase()` |
| `str.toLowerCase()` | `str.lowercase()` |
| `'value: ' + x` | `"value: $x"` or `"value: ${x}"` |

## Object Operations

| JavaScript | Kotlin |
|------------|--------|
| `{ x: 1, y: 2 }` | `Point(x = 1, y = 2)` (use data class) |
| `obj.x` | `obj.x` |
| `obj['key']` | `obj["key"]` (if map) |
| `{ ...obj, y: 3 }` | `obj.copy(y = 3)` (data class copy) |
| `Object.keys(obj)` | `obj.keys` (if map) |
| `Object.values(obj)` | `obj.values` (if map) |

## Exceptions

| JavaScript | Kotlin |
|------------|--------|
| `throw new Error('msg')` | `throw Exception("msg")` or custom exception |
| `try { ... } catch (e) { ... }` | `try { ... } catch (e: Exception) { ... }` |
| `try { ... } finally { ... }` | `try { ... } finally { ... }` |

## Common Patterns

### Pattern 1: Bitmap Access (CRITICAL!)

```javascript
// JavaScript - row-major [y][x]
if (b.data[y][x]) { ... }
b.data[y][x] = true;
for (let y = 0; y < height; y++) {
  for (let x = 0; x < width; x++) {
    if (b.data[y][x]) process();
  }
}
```

```kotlin
// Kotlin - column-major get(x, y)
if (b.get(x, y) == true) { ... }
b.set(x, y, true)
for (y in 0 until height) {
  for (x in 0 until width) {
    if (b.get(x, y) == true) process()
  }
}
```

**Remember**: x and y are SWAPPED!

### Pattern 2: Point Operations (Use Operator Overloading)

```javascript
// JavaScript - helper functions
const pointAdd = (a, b) => ({ x: a.x + b.x, y: a.y + b.y });
const pointSub = (a, b) => ({ x: a.x - b.x, y: a.y - b.y });
const pointNeg = (p) => ({ x: -p.x, y: -p.y });
const p3 = pointAdd(p1, p2);
```

```kotlin
// Kotlin - operator overloading (already defined in Types.kt)
operator fun Point.plus(other: Point) = Point(x + other.x, y + other.y)
operator fun Point.minus(other: Point) = Point(x - other.x, y - other.y)
operator fun Point.unaryMinus() = Point(-x, -y)
val p3 = p1 + p2
```

### Pattern 3: Null Safety

```javascript
// JavaScript - undefined checking
let x: number | undefined;
if (x !== undefined) { use(x); }
const y = x !== undefined ? x : defaultValue;
```

```kotlin
// Kotlin - null safety
var x: Int? = null
if (x != null) { use(x) }
// or
x?.let { use(it) }
// Elvis operator
val y = x ?: defaultValue
```

### Pattern 4: Error Handling

```javascript
// JavaScript
if (!condition) throw new Error('Invalid state');
```

```kotlin
// Kotlin - use specific exception types
if (!condition) throw QRDecodingException("Invalid state")
// or use require/check
require(condition) { "Invalid state" }
check(condition) { "Invalid state" }
```

### Pattern 5: Loop with Multiple Variables

```javascript
// JavaScript
for (let i = 0, j = 0; i < n; i++, j += 2) {
  process(i, j);
}
```

```kotlin
// Kotlin - use separate variable
var j = 0
for (i in 0 until n) {
  process(i, j)
  j += 2
}
```

### Pattern 6: Ternary Chains

```javascript
// JavaScript
const x = a ? 1 : b ? 2 : c ? 3 : 4;
```

```kotlin
// Kotlin - use when expression
val x = when {
  a -> 1
  b -> 2
  c -> 3
  else -> 4
}
```

### Pattern 7: Population Count

```javascript
// JavaScript
function popcnt(a) {
  let cnt = 0;
  while (a) {
    if (a & 1) cnt++;
    a >>= 1;
  }
  return cnt;
}
```

```kotlin
// Kotlin
private fun popcnt(a: Int): Int {
  var n = a
  var cnt = 0
  while (n != 0) {
    if (n and 1 != 0) cnt++
    n = n shr 1
  }
  return cnt
}
```

**Note**: Variable `a` becomes `n` because parameters are read-only in Kotlin.

### Pattern 8: Array Initialization with Function

```javascript
// JavaScript
const arr = new Array(n).fill(0).map((_, i) => i * 2);
```

```kotlin
// Kotlin
val arr = IntArray(n) { i -> i * 2 }
// or
val arr = Array(n) { i -> i * 2 }
```

### Pattern 9: String Encoding (ECI Example)

```javascript
// JavaScript
const eciToEncoding = { 1: 'iso-8859-1', 26: 'utf-8', 20: 'shift-jis' };
function decodeWithEci(bytes: Uint8Array, eci: number = 26): string {
  const encoding = eciToEncoding[eci];
  if (!encoding) throw new Error(`Unsupported ECI: ${eci}`);
  return new TextDecoder(encoding).decode(bytes);
}
```

```kotlin
// Kotlin
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

**Import needed**: `java.nio.charset.Charset`

### Pattern 10: Bitmap Negate (Inverted QR Example)

```javascript
// JavaScript
class Bitmap {
  negate() {
    for (let y = 0; y < this.height; y++) {
      for (let x = 0; x < this.width; x++) {
        this.data[y][x] = !this.data[y][x];
      }
    }
  }
}

// Usage with fallback
try {
  ({ bl, tl, tr } = findFinder(b));
} catch (e) {
  try {
    b.negate();
    ({ bl, tl, tr } = findFinder(b));
  } catch (e2) {
    b.negate(); // undo
    throw e;
  }
}
```

```kotlin
// Kotlin
class Bitmap {
  fun negate() {
    for (y in 0 until height) {
      for (x in 0 until width) {
        set(x, y, get(x, y)?.not())  // Note: x, y order!
      }
    }
  }
}

// Usage with fallback
val (bl, tl, tr) = try {
  findFinder(b)
} catch (e: Exception) {
  try {
    b.negate()
    findFinder(b)
  } catch (e2: Exception) {
    b.negate() // undo
    throw e
  }
}
```

## Kotlin Coding Style Guidelines

These conventions are used throughout the Kotlin QR library. **ALWAYS follow these when porting code**.

### 1. Use `object` for Singletons

```kotlin
// JavaScript: module-level functions
export function decode(img) { ... }

// Kotlin: object singleton
object QRDecoder {
  fun decode(img: Image): String { ... }
}
```

### 2. Use `data class` for Value Objects

```kotlin
data class Point(val x: Double, val y: Double)
data class Pattern(val x: Double, val y: Double, val moduleSize: Double)
```

**Benefits**: Automatic equals(), hashCode(), toString(), copy()

### 3. Prefer `val` over `var`

```kotlin
// Immutable by default
val width = 640
val height = 480

// Only use var when necessary
var retries = 0
while (retries < 3) {
  retries++
}
```

### 4. Use `when` Expressions

```kotlin
// Instead of if-else chains
val mode = when (bits) {
  "0001" -> EncodingType.NUMERIC
  "0010" -> EncodingType.ALPHANUMERIC
  "0100" -> EncodingType.BYTE
  else -> throw InvalidFormatException("Unknown mode: $bits")
}
```

### 5. Use Ranges

```kotlin
// Instead of C-style loops
for (i in 0 until n) { ... }        // 0 to n-1
for (i in 0..n) { ... }             // 0 to n (inclusive)
for (i in n downTo 0) { ... }       // n to 0
for (i in 0 until n step 2) { ... } // 0, 2, 4, ...
```

### 6. Use Extension Functions

```kotlin
// Add methods to existing types
fun Point.distanceTo(other: Point): Double {
  val dx = x - other.x
  val dy = y - other.y
  return sqrt(dx * dx + dy * dy)
}

// Usage
val dist = p1.distanceTo(p2)
```

### 7. Use Operator Overloading

```kotlin
operator fun Point.plus(other: Point) = Point(x + other.x, y + other.y)
operator fun Point.minus(other: Point) = Point(x - other.x, y - other.y)
operator fun Point.unaryMinus() = Point(-x, -y)

// Usage
val p3 = p1 + p2
val p4 = -p1
```

### 8. Use Single-Expression Functions

```kotlin
// Instead of
fun double(x: Int): Int {
  return x * 2
}

// Use
fun double(x: Int) = x * 2
```

### 9. Use KDoc for Public APIs

```kotlin
/**
 * Decode a QR code from an image.
 *
 * @param image The image containing the QR code
 * @return The decoded string content
 * @throws QRDecodingException if decoding fails
 */
fun decode(image: Image): String { ... }
```

### 10. Use Destructuring

```kotlin
val (bl, tl, tr) = findFinder(b)
val (x, y) = point
```

## Common Pitfalls

### 1. Array Index Order (MOST COMMON BUG!)

❌ **Wrong**:
```kotlin
if (b.get(y, x) == true) { ... }  // WRONG!
```

✅ **Correct**:
```kotlin
if (b.get(x, y) == true) { ... }  // Correct - x first!
```

### 2. Integer Division

```javascript
// JavaScript - always returns float
const avg = (a + b) / 2;  // 5 / 2 = 2.5
```

```kotlin
// Kotlin - integer division for Int / Int
val avg = (a + b) / 2  // 5 / 2 = 2 (truncated!)

// Use .toDouble() for float division
val avg = (a + b).toDouble() / 2  // 5 / 2 = 2.5
```

### 3. Unsigned Right Shift

```javascript
// JavaScript
const x = -1 >>> 1;  // Unsigned right shift
```

```kotlin
// Kotlin - use ushr
val x = -1 ushr 1  // Unsigned right shift
```

### 4. Array Initialization

```javascript
// JavaScript - creates array with undefined elements
const arr = new Array(10);  // [undefined, undefined, ...]
```

```kotlin
// Kotlin - MUST provide initializer
val arr = Array(10) { 0 }  // [0, 0, 0, ...]
// or
val arr = IntArray(10)  // [0, 0, 0, ...]
```

### 5. Loop Bounds

```javascript
// JavaScript - exclusive end
for (let i = 0; i < n; i++) { ... }
```

```kotlin
// Kotlin
for (i in 0 until n) { ... }  // Correct - until is exclusive
for (i in 0..n-1) { ... }     // Also correct
for (i in 0..n) { ... }       // WRONG - includes n!
```

### 6. ByteArray vs Uint8Array

JavaScript `Uint8Array` is unsigned (0-255), but Kotlin `ByteArray` is signed (-128 to 127).

```kotlin
// When working with bytes that might be > 127
val unsigned = byte.toInt() and 0xFF  // Convert to unsigned
```

### 7. Mutable Parameters

```javascript
// JavaScript - parameters are mutable
function foo(x) {
  x = x + 1;  // OK
  return x;
}
```

```kotlin
// Kotlin - parameters are read-only
fun foo(x: Int): Int {
  // x = x + 1  // ERROR! Parameters are val
  val y = x + 1  // Use local variable
  return y
}
```

### 8. Null vs Undefined

JavaScript has both `null` and `undefined`. Kotlin only has `null`.

```javascript
// JavaScript
if (x !== undefined) { ... }
```

```kotlin
// Kotlin
if (x != null) { ... }
```

## Imports Needed

Common imports for Kotlin QR code:

```kotlin
import kotlin.math.sqrt
import kotlin.math.floor
import kotlin.math.ceil
import kotlin.math.abs
import kotlin.math.min
import kotlin.math.max
import java.nio.charset.Charset  // For ECI encoding
```

## Quick Reference Card

| Task | JavaScript | Kotlin |
|------|------------|--------|
| Loop 0 to n-1 | `for (let i = 0; i < n; i++)` | `for (i in 0 until n)` |
| Bitmap get | `b.data[y][x]` | `b.get(x, y)` |
| Bitmap set | `b.data[y][x] = val` | `b.set(x, y, val)` |
| Bitwise AND | `a & b` | `a and b` |
| Bitwise OR | `a \| b` | `a or b` |
| Bitwise XOR | `a ^ b` | `a xor b` |
| Right shift | `a >> b` | `a shr b` |
| Unsigned shift | `a >>> b` | `a ushr b` |
| Ternary | `a ? b : c` | `if (a) b else c` |
| Null check | `x !== undefined` | `x != null` |
| Default value | `x \|\| default` | `x ?: default` |
| String template | `'value: ' + x` | `"value: $x"` |
| Array map | `arr.map(x => ...)` | `arr.map { x -> ... }` |
| Throw error | `throw new Error(msg)` | `throw Exception(msg)` |
