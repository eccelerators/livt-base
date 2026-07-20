# Livt.Base

The base library for the Livt programming language.

`Livt.Base` is part of the official Livt standard library. It provides
foundational components that other Livt packages and application code depend on.
The library is small, dependency-free, and intended to become the stable
foundation for other packages. The API is still stabilizing while the Livt
compiler and VHDL backend mature. Every component is either fully synthesizable
or explicitly designated as simulation and test tooling.

## 📦 Package

```toml
[dependencies]
Livt.Base = "0.2.0"
```

## 📚 Namespaces

`Livt.Base` publishes a small set of foundational root namespaces rather than a
single `Livt.Base.*` namespace. This keeps common utilities short at call sites
while the package itself remains the dependency boundary.

| Namespace | Component | Synthesizable | Purpose |
|---|---|---|---|
| `Livt.Bits` | `Bits` | Yes | Bit access, masks, field extraction and insertion, population count, rotation, and extension |
| `Livt.Ascii` | `Ascii` | Yes | ASCII constants, character classification, and character conversion |
| `Livt.Convert` | `Convert` | Yes | Explicit integer width conversions with named overflow semantics |
| `Livt.Format` | `Format` | Yes | Fixed-width string formatting for byte, int, and uint values |
| `Livt.Array` | `Array` | Yes | Fixed-size array operations: bounds checking, byte equality, and byte search |
| `Livt.String` | `StringHelper` | Yes | Byte-array string search: starts-with, ends-with, contains, and substring index |
| `Livt.Timing` | `PeriodicTickProvider`, fixed-period providers | Yes | Context-derived periodic one-tick pulses |
| `Livt.Diagnostics` | `Assert` | No | Assertion and reporting components for test and simulation use |

## ⏱️ `Livt.Timing`

Provides synthesizable periodic pulse components. Each provider asserts its
`tick` output for one context tick whenever its interval expires. Fixed-period
providers derive their tick counts from the component context, so their
real-time period remains stable when the component is used at another clock
frequency.

```livt
using Livt.Timing

tick1ms: logic
tick200ms: logic
millisecondProvider: PeriodicTickProvider1ms
applicationProvider: PeriodicTickProvider

new()
{
    this.millisecondProvider = new PeriodicTickProvider1ms(this.tick1ms)
    var period = this.context.TicksFor(200ms)
    this.applicationProvider = new PeriodicTickProvider(period, this.tick200ms)
}
```

| Component | Period |
|---|---:|
| `PeriodicTickProvider` | Configured in context ticks |
| `PeriodicTickProvider1us` | 1 microsecond |
| `PeriodicTickProvider1ms` | 1 millisecond |
| `PeriodicTickProvider10ms` | 10 milliseconds |
| `PeriodicTickProvider100ms` | 100 milliseconds |
| `PeriodicTickProvider1s` | 1 second |

`PeriodicTickProvider` requires an interval of at least one context tick. Use
`GetPeriodTicks()` when diagnostics or tests need to inspect the resolved
interval. `TicksFor(...)` always rounds a positive duration up to at least one
tick.

## 🔗 Dependency Policy

`Livt.Base` has no dependencies on other Livt packages. It is the foundation
that other packages build on.

- `livt-math`, `livt-net`, `livt-ml`, and domain-specific packages may depend
  on `Livt.Base`.
- `Livt.Base` does not contain numeric algorithms, networking, machine learning,
  vendor-specific functionality, or protocol implementations. Those belong in
  their respective packages.

## 🧪 Build and Test

```sh
livt test
```

To force full regeneration without removing dependencies:

```sh
rm -rf out .livt/src.json .livt/ghdl
livt test
```

---

## 🔢 `Livt.Bits`

Provides bit and logic-vector operations for non-negative integer values. All
functions treat the value as a non-negative integer stored in `int`. Bit indices
are 0-based (0 = least significant bit). All functions are synthesizable.

### Known Synthesis Costs

Several generic `Bits` helpers compute powers of two at runtime and then use
division or modulo with variable divisors. The generated VHDL is correct in the
current tests, but the compiler reports variable-divider warnings for functions
such as `GetBit`, `SetBit`, `ClearBit`, `ToggleBit`, `Extract`, `RotateLeft`,
`RotateRight`, and `SignExtend`.

These helpers are useful portable reference utilities. Timing-critical FPGA
designs should prefer fixed-width, constant-index logic or future specialized
helpers where the shift/mask structure is statically known.

```livt
using Livt.Bits

var bits: Bits = new Bits()

bits.GetBit(0b1010, 1)                   // 1
bits.SetBit(0b1010, 0)                   // 0b1011
bits.ClearBit(0b1010, 3)                 // 0b0010
bits.ToggleBit(0b1010, 0)               // 0b1011
bits.WithBit(0b1010, 1, 0)              // 0b1000

bits.Mask(4)                             // 0b1111 = 15
bits.MaskRange(5, 2)                     // 0b111100 = 60
bits.Extract(0b11010110, 5, 2)           // 0b0101 = 5
bits.Insert(0b11000000, 5, 2, 0b0101)   // 0b11010100

bits.CountOnes(0b10110101)               // 5
bits.CountZeros(0b10110101, 8)           // 3
bits.Parity(0b10110101)                  // 1
bits.ReverseBits(0b10110001, 8)          // 0b10001101

bits.RotateLeft(0b10110001, 2, 8)        // 0b11000110
bits.RotateRight(0b10110001, 2, 8)       // 0b01101100
bits.ZeroExtend(0b1111, 4, 8)            // 0b00001111
bits.SignExtend(0b1111, 4, 8)            // 0b11111111
```

### API

| Function | Description |
|---|---|
| `GetBit(value, index)` | Returns the bit at `index` as 0 or 1 |
| `SetBit(value, index)` | Returns `value` with the bit at `index` set to 1 |
| `ClearBit(value, index)` | Returns `value` with the bit at `index` cleared to 0 |
| `ToggleBit(value, index)` | Returns `value` with the bit at `index` inverted |
| `WithBit(value, index, bit)` | Returns `value` with the bit at `index` set to `bit` (0 or 1) |
| `Mask(width)` | Returns an integer with the lowest `width` bits set to 1 |
| `MaskRange(high, low)` | Returns an integer with bits `low` through `high` set to 1 |
| `Extract(value, high, low)` | Extracts bits `low` through `high` and shifts the result to bit 0 |
| `Insert(value, high, low, field)` | Returns `value` with bits `low` through `high` replaced by `field` |
| `CountOnes(value)` | Returns the number of 1-bits in the lower 31 bits of `value` |
| `CountZeros(value, width)` | Returns the number of 0-bits in the lower `width` bits of `value` |
| `Parity(value)` | Returns 1 if the number of 1-bits is odd, otherwise 0 |
| `ReverseBits(value, width)` | Returns `value` with its lower `width` bits in reverse order |
| `RotateLeft(value, amount, width)` | Rotates `value` left by `amount` positions within a `width`-bit field |
| `RotateRight(value, amount, width)` | Rotates `value` right by `amount` positions within a `width`-bit field |
| `ZeroExtend(value, fromWidth, toWidth)` | Zero-extends `value` from `fromWidth` to `toWidth` bits |
| `SignExtend(value, fromWidth, toWidth)` | Sign-extends `value` from `fromWidth` to `toWidth` bits |
| `Pow2(n)` | Returns $2^n$ for `n` in `[0, 30]` |
| `RequiredBits(value)` | Returns the minimum number of bits to represent `value`; `RequiredBits(0)` returns 1 |

---

## 🔤 `Livt.Ascii`

Provides ASCII constants, character classification functions, and character
conversion functions. All functions operate on `byte` values and are
synthesizable. Non-ASCII input bytes (0x80–0xFF) return `false` from
classification functions and are returned unchanged by conversion functions.

```livt
using Livt.Ascii

var ascii: Ascii = new Ascii()

Ascii.LINE_FEED           // 0x0A
Ascii.SPACE               // 0x20
Ascii.DIGIT_0             // 0x30
Ascii.UPPER_A             // 0x41
Ascii.LOWER_A             // 0x61

ascii.IsDigit(0x35)       // true  — '5'
ascii.IsHexDigit(0x41)    // true  — 'A'
ascii.IsLetter(0x41)      // true  — 'A'
ascii.IsWhitespace(0x20)  // true  — ' '
ascii.IsPrintable(0x7E)   // true  — '~'
ascii.IsControl(0x0A)     // true  — LF

ascii.ToUpper(0x61)           // 0x41
ascii.ToLower(0x41)           // 0x61
ascii.DigitValue(0x35)        // 5
ascii.HexValue(0x41)          // 10
ascii.FromDigit(7)            // 0x37  — '7'
ascii.FromHexDigit(10, true)  // 0x41  — 'A'
ascii.FromHexDigit(10, false) // 0x61  — 'a'
```

### Constants

`NUL`, `BACKSPACE`, `TAB`, `LINE_FEED`, `CARRIAGE_RETURN`, `ESCAPE`, `DELETE`,
`SPACE`, `EXCLAMATION`, `QUOTE`, `HASH`, `PERCENT`, `AMPERSAND`, `LEFT_PAREN`,
`RIGHT_PAREN`, `ASTERISK`, `PLUS`, `COMMA`, `MINUS`, `DOT`, `SLASH`, `COLON`,
`SEMICOLON`, `LESS_THAN`, `EQUALS`, `GREATER_THAN`, `QUESTION`, `AT`,
`LEFT_BRACKET`, `BACKSLASH`, `RIGHT_BRACKET`, `CARET`, `UNDERSCORE`,
`BACKTICK`, `LEFT_BRACE`, `PIPE`, `RIGHT_BRACE`, `TILDE`, `DIGIT_0`,
`DIGIT_9`, `UPPER_A`, `UPPER_F`, `UPPER_Z`, `LOWER_A`, `LOWER_F`, `LOWER_Z`.

### API

| Function | Description |
|---|---|
| `IsDigit(c)` | True if `c` is `'0'..'9'` |
| `IsHexDigit(c)` | True if `c` is `'0'..'9'`, `'A'..'F'`, or `'a'..'f'` |
| `IsBinaryDigit(c)` | True if `c` is `'0'` or `'1'` |
| `IsOctalDigit(c)` | True if `c` is `'0'..'7'` |
| `IsUpper(c)` | True if `c` is `'A'..'Z'` |
| `IsLower(c)` | True if `c` is `'a'..'z'` |
| `IsLetter(c)` | True if `c` is `'A'..'Z'` or `'a'..'z'` |
| `IsAlphaNumeric(c)` | True if `c` is a letter or a decimal digit |
| `IsWhitespace(c)` | True if `c` is space, tab, line feed, or carriage return |
| `IsPrintable(c)` | True if `c` is in `0x20..0x7E` |
| `IsControl(c)` | True if `c` is in `0x00..0x1F` or `0x7F` |
| `ToUpper(c)` | Returns the uppercase form of `c`; non-lowercase characters are unchanged |
| `ToLower(c)` | Returns the lowercase form of `c`; non-uppercase characters are unchanged |
| `DigitValue(c)` | Returns the numeric value of `'0'..'9'`; returns `0xFF` for non-digit input |
| `HexValue(c)` | Returns the numeric value 0–15 of a hex digit; returns `0xFF` for invalid input |
| `FromDigit(value)` | Returns the ASCII character for `value` in 0–9; returns `'?'` for out-of-range input |
| `FromHexDigit(value, uppercase)` | Returns the ASCII character for `value` in 0–15; uppercase or lowercase as specified |

---

## 🔄 `Livt.Convert`

Provides explicit integer width conversion functions. All functions are
synthesizable. The name of each function makes the overflow behavior visible at
the call site.

| Suffix | Behavior when the input is outside the target range |
|---|---|
| `Clamped` | Clamps to the nearest boundary of the target range |
| `Wrapping` | Wraps modulo the target type range |
| `Truncated` | Discards high bits; equivalent to `Wrapping` for unsigned targets |

```livt
using Livt.Convert

var conv: Convert = new Convert()

conv.ToUInt8Clamped(300)     // 255
conv.ToUInt8Clamped(-5)      // 0
conv.ToUInt8Wrapping(260)    // 4
conv.ToInt8Clamped(200)      // 127
conv.ToInt8Clamped(-200)     // -128
conv.ToInt8Wrapping(130)     // -126
conv.ToUInt16Clamped(70000)  // 65535
conv.ToInt16Clamped(-40000)  // -32768
conv.ToByteClamped(300)      // 255
conv.ToByteWrapping(0x1FF)   // 255
conv.ToByteTruncated(0x1FF)  // 255
```

### API

| Function | Range | Overflow behavior |
|---|---|---|
| `ToUInt8Clamped(value)` | [0, 255] | Clamp |
| `ToUInt8Wrapping(value)` | [0, 255] | Wrap mod 256 |
| `ToUInt8Truncated(value)` | [0, 255] | Keep low 8 bits |
| `ToInt8Clamped(value)` | [-128, 127] | Clamp |
| `ToInt8Wrapping(value)` | [-128, 127] | Wrap mod 256, signed |
| `ToInt8Truncated(value)` | [-128, 127] | Keep low 8 bits, signed |
| `ToUInt16Clamped(value)` | [0, 65535] | Clamp |
| `ToUInt16Wrapping(value)` | [0, 65535] | Wrap mod 65536 |
| `ToUInt16Truncated(value)` | [0, 65535] | Keep low 16 bits |
| `ToInt16Clamped(value)` | [-32768, 32767] | Clamp |
| `ToInt16Wrapping(value)` | [-32768, 32767] | Wrap mod 65536, signed |
| `ToInt16Truncated(value)` | [-32768, 32767] | Keep low 16 bits, signed |
| `ToByteClamped(value)` | [0, 255] | Clamp — `byte` is Livt's native unsigned 8-bit type |
| `ToByteWrapping(value)` | [0, 255] | Wrap mod 256 |
| `ToByteTruncated(value)` | [0, 255] | Keep low 8 bits |

---

## 🧾 `Livt.Format`

Provides fixed-width string formatting helpers for machine values. Decimal
output is zero-padded; signed integer output uses a leading sign. All functions
are synthesizable.

```livt
using Livt.Format

var fmt: Format = new Format()

fmt.ToStringByte(7)      // "007"
fmt.ToStringInt(-42)     // "-0000000042"
fmt.ToStringUInt(42)     // "0000000042"
fmt.ToHexByte(0xAF)      // "AF"
fmt.ToBinaryByte(0xA5)   // "10100101"
```

### API

| Function | Return | Description |
|---|---|---|
| `ToStringByte(value)` | `string[3]` | Formats a byte as three decimal digits |
| `ToStringInt(value)` | `string[11]` | Formats an int as a sign plus ten decimal digits |
| `ToStringUInt(value)` | `string[10]` | Formats a uint as ten decimal digits |
| `ToHexByte(value)` | `string[2]` | Formats a byte as two uppercase hexadecimal digits |
| `ToBinaryByte(value)` | `string[8]` | Formats a byte as eight binary digits |

`logic` values should be cast explicitly before formatting so the caller chooses
the intended interpretation:

```livt
fmt.ToStringUInt(bits as uint)
fmt.ToStringInt(bits as int)
fmt.ToHexByte(bits8 as byte)
```

`ToStringInt` supports signed values greater than the minimum signed int. The
minimum value is excluded because the implementation negates negative values
before extracting decimal digits.

---

## 📋 `Livt.Array`

Provides fixed-size array operations that fit Livt's synthesis model. Arrays
must have a statically known size. All functions are synthesizable.

The `byte[]` unsized parameter form is supported: any `byte[N]` can be passed
to a `byte[]` parameter. The size must be supplied as a separate `int`
parameter; the caller is responsible for providing the correct value.

```livt
using Livt.Array

var arr: Array = new Array()
var data: byte[4] = [0x10, 0x20, 0x30, 0x20]
var copy: byte[4] = [0x10, 0x20, 0x30, 0x20]

arr.IsIndexInRange(0, 4)     // true
arr.IsIndexInRange(4, 4)     // false
arr.Equals(data, copy, 4)    // true
arr.IndexOf(data, 4, 0x20)   // 1
arr.CountOf(data, 4, 0x20)   // 2
```

### API

| Function | Description |
|---|---|
| `IsIndexInRange(index, size)` | True if `index` is within `[0, size-1]` |
| `Equals(a, b, size)` | True if the first `size` bytes of `a` and `b` are equal |
| `IndexOf(arr, size, value)` | Returns the first index of `value` in the first `size` bytes; -1 if not found |
| `CountOf(arr, size, value)` | Returns the number of occurrences of `value` in the first `size` bytes |

---

## 🛠️ `Livt.Diagnostics`

Provides assertion components for test and simulation use. This namespace is
**not synthesizable** and is intended for `@Test` components only.

`Assert` wraps the built-in `assert` statement with named functions that
communicate intent at the call site.

```livt
using Livt.Diagnostics

@Test
component ExampleTest
{
    chk: Assert

    new()
    {
        this.chk = new Assert()
    }

    @Test
    fn ValueIsInRange()
    {
        this.chk.AssertIntInRange(5, 0, 10)
        this.chk.AssertByteEqual(0xFF, 0xFF)
        this.chk.AssertTrue(true)
    }
}
```

### API

| Function | Description |
|---|---|
| `AssertTrue(value)` | Fails if `value` is false |
| `AssertFalse(value)` | Fails if `value` is true |
| `AssertBoolEqual(expected, actual)` | Fails if the two boolean values differ |
| `AssertIntEqual(expected, actual)` | Fails if the two integer values differ |
| `AssertIntNotEqual(a, b)` | Fails if the two integer values are equal |
| `AssertIntGreater(value, threshold)` | Fails if `value` is not strictly greater than `threshold` |
| `AssertIntLess(value, threshold)` | Fails if `value` is not strictly less than `threshold` |
| `AssertIntInRange(value, low, high)` | Fails if `value` is outside the inclusive range `[low, high]` |
| `AssertByteEqual(expected, actual)` | Fails if the two byte values differ |
| `AssertByteNotEqual(a, b)` | Fails if the two byte values are equal |
| `ReportHex(value)` | Reports the 8-digit lowercase hex representation of `value` to the simulation log. Works for non-negative values `[0, 2147483647]`. |
| `ReportValue(value)` | Reports `value` as a decimal integer to the simulation log |
| `ReportBinary(value)` | Reports `value` as a 32-bit binary string to the simulation log |

---

## 📝 `Livt.String`

Provides byte-oriented string search and comparison helpers. Strings in Livt
are most conveniently handled as encoded byte arrays — use `"text".Encode()`
to convert a literal to bytes and `byteArray.Decode()` to convert back.

```livt
using Livt.String

var str: StringHelper = new StringHelper()
var buf: byte[128]  // received protocol data
var method: byte[3] = "GET".Encode()

if (str.StartsWith(buf, 128, method, 3))
{
    // handle GET
}
```

### API

| Function | Description |
|---|---|
| `StartsWith(data, dataSize, prefix, prefixSize)` | Returns `true` if the first `prefixSize` bytes of `data` equal `prefix` |
| `EndsWith(data, dataSize, suffix, suffixSize)` | Returns `true` if the last `suffixSize` bytes of `data` equal `suffix` |
| `Contains(data, dataSize, sub, subSize)` | Returns `true` if `sub` occurs anywhere within `data` |
| `IndexOf(data, dataSize, sub, subSize)` | Returns the index of the first occurrence of `sub` in `data`, or `-1` if not found |
| `LastIndexOf(data, dataSize, sub, subSize)` | Returns the index of the last occurrence of `sub` in `data`, or `-1` if not found |
| `EqualsAt(data, dataSize, pos, sub, subSize)` | Returns `true` if `sub` matches `data` starting at position `pos` |
| `EqualsIgnoreCase(a, aSize, b, bSize)` | Returns `true` if `a` and `b` are equal, ignoring ASCII letter case |
| `NullTerminatedLength(data, maxSize)` | Returns the number of bytes before the first `0x00`, or `maxSize` if none found |
| `ParseDecimalUInt(data, dataSize)` | Parses `'0'`–`'9'` ASCII bytes as an unsigned integer; returns `-1` on invalid input |
| `ParseHexUInt(data, dataSize)` | Parses `'0'`–`'9'`, `'a'`–`'f'`, `'A'`–`'F'` bytes as an unsigned integer; returns `-1` on invalid input |

---

## ⚙️ Synthesis Classification

| Namespace | Synthesizable |
|---|---|
| `Livt.Bits` | Yes |
| `Livt.Ascii` | Yes |
| `Livt.Convert` | Yes |
| `Livt.Format` | Yes |
| `Livt.Array` | Yes |
| `Livt.String` | Yes — byte-array helpers are synthesizable |
| `Livt.Timing` | Yes — periodic pulses are derived from the component context |
| `Livt.Diagnostics` | No — simulation and test tooling only |

---

## 🚀 Versioning

`Livt.Base` follows semantic versioning. The current version is `0.2.0`. The
API is stabilizing. Dependent packages should pin the version explicitly.

## 📄 License

This project is licensed under the MIT License. See [LICENSE](LICENSE).
