# COMPILER.md — Livt.Base Compiler Observations

This file records compiler quirks, VHDL-generation issues, and confirmed
workarounds discovered while implementing `livt-base`.

---

## §1 — Variable Redeclaration Scope

**Symptom:** Compiler error `The variable 'x' is declared multiple times.`

**Cause:** Livt variable declarations (`var x: type = ...`) have function-level
scope, not block-level scope.  Two `if` branches in the same function cannot
both declare `var x`.

**Workaround:** Use unique names per branch, e.g. `vi`, `viU`, `viL` for an
uppercase and lowercase branch.

---

## §2 — VHDL Reserved Word Collision in Constant Names

**Symptom:** GHDL parse error such as:
```
constant NULL : std_logic_vector(7 downto 0) := x"00";
         ^
an identifier is expected instead of 'null'
```

**Cause:** Livt constant names are emitted verbatim into VHDL `constant`
declarations.  VHDL has reserved words that differ from Livt identifiers.
`NULL` is a VHDL reserved word.

**Known reserved words to avoid:** `NULL`, `SIGNAL`, `PROCESS`, `TYPE`,
`PORT`, `MAP`, `WAIT`, `IN`, `OUT`.

**Workaround:** Use `NUL` for the ASCII null character (the correct ASCII
abbreviation), or prefix constants with a context letter, e.g. `ASCII_NULL`.

---

## §3 — Calling the Same Private Function Multiple Times in One Expression

**Symptom:** Incorrect simulation results when a function calls the same
private helper function multiple times in one expression, such as:
```livt
return this.F(a) + this.F(b) + this.F(c) + this.F(d)
```

**Cause:** The Livt VHDL code generator allocates a single shared return-value
variable per (caller, callee) pair.  Every call to `F` overwrites this
variable.  When the final return expression is generated, only the last stored
value is used for all operands, producing wrong results.

**Example failure observed:** `CountOnes(1)` returned 0 because the compiler
emitted:
```vhdl
return_value <= last_result + last_result + last_result + last_result;
```
instead of summing the four distinct results.

**Workaround:** Do not call the same private function multiple times in a
single Livt function body.  Either inline the logic using a `for` loop, or if
multiple calls are genuinely needed, confirm that **each result is saved to a
distinct local variable** before the next call:

```livt
// Safe — but verify the compiler generates separate variables
var c0: int = this.F(a)  // saved before F(b) runs
var c1: int = this.F(b)
```

The safest pattern is to inline repeated logic as a loop rather than calling
the helper multiple times.

---

## §4 — `assert` in Non-Test Component Functions

**Status:** Confirmed working.

`assert` statements can appear inside ordinary (non-`@Test`) component function
bodies.  The compiler emits standard VHDL assertion statements, which trigger
during simulation exactly as they would in `@Test fn` bodies.

This means `Livt.Diagnostics` assertion helpers (e.g. `AssertTrue(value)`)
that internally contain `assert value == true` work correctly when called from
`@Test fn` functions in test components.

---

## §5 — Computed Expression as Private Function Argument

**Status:** Suspected risk based on CONTEXT.md recommendations; not yet
confirmed to fail in `livt-base`.

**Symptom:** Possible wrong argument passed to function.

**Pattern to avoid:**
```livt
var ones: int = this.CountOnes(value % this.Pow2(width))
```

**Safe pattern:**
```livt
var p: int = this.Pow2(width)
var masked: int = value % p
var ones: int = this.CountOnes(masked)
```

Always bind computed intermediate values to local variables before passing them
as function arguments.

---

## §6 — Unsized Array Parameters (`byte[]`, `int[]`)

**`byte[]` (unsized byte array) parameters: Confirmed working.**

A function may declare a parameter as `byte[]` (no explicit size) and a caller
can pass any `byte[N]` argument.  Element reads inside the function (`arr[i]`)
generate correct VHDL and work at simulation.  The size must be supplied as a
separate `int` parameter; the compiler does not inject a length automatically.

```livt
// Works: caller can pass byte[16], byte[32], byte[64], etc.
fn ByteIndexOf(arr: byte[], size: int, value: byte) int
{
    for (var i = 0; i < size; i++)
    {
        var ai: byte = arr[i]
        if (ai == value) { return i }
    }
    return -1
}
```

**`int[]` (unsized int array) parameters: NOT supported.**

The compiler accepts the syntax but generates invalid VHDL:
```vhdl
variable result : integer(new_size - 1 downto 0) := (others => 0);
-- error: scalar types may only be constrained by range
```

Use `int[N]` with an explicit size for integer array parameters instead.

---

## §7 — `Simulation.Report` String Variable Support

**Status: Fixed in compiler (2026-05-17).**

Previously crashed with:
```
class com.eccelerators.livt.references.VariableReference cannot be cast to
class com.eccelerators.livt.literals.StringLiteral
```

After the fix, both string literals and string variables work:
```livt
var msg: string = "hello"
Simulation.Report(msg)       // now works
Simulation.Report("hello")   // also works
```

Note: `string` variables obtained by decoding a `byte[]` also work:
```livt
var encoded: byte[5] = "Hello".Encode()
var decoded: string = encoded.Decode()
Simulation.Report(decoded)   // works
```

---

## §8 — `string` Type as Function Parameter Now Supported (Sized Variant)

**Status: Fixed in compiler (2026-05-17).**

Previously, using `string` as a function parameter caused a crash:
```
Cannot invoke "Object.getClass()" because "sourceType" is null
```

After the fix, sized string parameters work:
```livt
public fn IsHello(msg: in string[5]) bool  // works with in direction
```

Note: the `in` direction keyword is required for sized string parameters in
function signatures. Unsized `string[]` parameters remain untested.

---

## §9 — `& intLiteral` Compiles Incorrectly as VHDL `and '1'`

**Status:** Confirmed bug (2026-05-17).

**Symptom:** Bitwise AND with an integer literal (`value & 15`, `value & 0xF`)
compiles to VHDL `and '1'` (std_logic scalar) instead of masking with the
literal value.  This produces wrong results at simulation.

**Example failure:**
```livt
var nibble: int = x & 15   // should be x AND 0x0000000F
```

Generated VHDL:
```vhdl
nibble := to_integer(to_signed(x, 32) and '1');  -- WRONG: masks nothing meaningful
```

With `and '1'` in VHDL-2008, each bit of the signed value is ANDed with `'1'`,
leaving it unchanged — so the masking has no effect.  For large values this
produces nibbles that exceed the byte range, causing `to_unsigned` failures
at simulation time.

**Workaround:** Replace `& N` with `% (N+1)` when `N` is a power-of-2 minus 1
(i.e., a bit mask):

```livt
var nibble: int = x % 16     // correct workaround for x & 15
var bit: int = x % 2         // correct workaround for x & 1
```

For general masking where the mask is not a power-of-2 minus 1, bind the mask
to a `var` first; AND with a variable appears to work correctly:

```livt
var mask: int = 0x0F
var nibble: int = x & mask   // works when mask is a variable
```

---

## §10 — Passing `byte[]` Parameters Through to Private Functions

**Status:** Confirmed VHDL generation failure (2026-05-17).

**Symptom:** A VHDL compilation error:
```
no declaration for "data"
prefix must denote an array object or type
```

**Cause:** When a public function with `byte[]` (unsized) parameters calls a
private helper function that also takes `byte[]` parameters, the VHDL generator
tries to compute `data'LENGTH` and `sub'LENGTH` for the private function call
but those identifiers are not in scope at the generated call site.

**Pattern to avoid:**
```livt
// Public fn with byte[] params calling private fn with byte[] params
private fn MatchesAt(data: byte[], pos: int, sub: byte[], subSize: int) bool { ... }

public fn ByteContains(data: byte[], dataSize: int, sub: byte[], subSize: int) bool
{
    ...
    if (this.MatchesAt(data, i, sub, subSize) == true) { ... }  // FAILS
}
```

**Workaround:** Inline the private function's logic using nested loops.  Do not
pass `byte[]` parameters from a public function to a private helper function:

```livt
public fn ByteContains(data: byte[], dataSize: int, sub: byte[], subSize: int) bool
{
    var match: bool = false
    for (var i = 0; i < limit; i++)
    {
        match = true
        for (var j = 0; j < subSize; j++)
        {
            var idx: int = i + j
            var di: byte = data[idx]
            var si: byte = sub[j]
            if (di != si) { match = false }
        }
        if (match == true) { return true }
    }
    return false
}
```

---

## §11 — Local Path Dependency Compiles All `.lvt` Files in `tests/`

**Status:** Confirmed behaviour.

**Symptom:** When package A declares a local path dependency on package B
(`"Livt.Base" = { path = "../livt-base" }`), the Livt compiler compiles **all**
`.lvt` files it finds in B's `tests/` directory — not just those listed in B's
`livt.toml [tests] components`.

**Impact:** A broken or experimental test file in the dependency's `tests/`
directory will cause a compiler crash in the dependent package, even if that
file is not registered in the dependency's `livt.toml`.

**Workaround:** Keep every `.lvt` file in `tests/` compilable at all times.
Remove or fix experiment/scratch files before committing.  Do not leave broken
test cases in the `tests/` directory even if they are absent from `livt.toml`.

---

## §12 — Test Component File Name Must Match Component Name

**Status:** Confirmed behaviour.

**Symptom:** A test component is silently skipped — it does not appear in the
test run output and its `@Test fn` functions are never executed.  No error is
reported.

**Cause:** The Livt test runner maps test components to `.lvt` files by file
name.  The file name must match the component name exactly (case-sensitive).  A
file named `StringNamespaceProbeTest.lvt` containing `component StringHelperTest`
will not be compiled for the test run.

**Example failure:**
```
// File: StringNamespaceProbeTest.lvt
// Component inside: StringHelperTest
// → test is silently skipped
```

**Workaround:** Rename the file so it matches the component name:
```
// File: StringHelperTest.lvt
// Component inside: StringHelperTest
// → test runs correctly
```

---

## §13 — `0 as byte` Generates VHDL Character Literal `'0'`

**Status:** Confirmed VHDL type error (2026-05-17).

**Symptom:** A VHDL compilation error:
```
no function declarations for operator "="
    if nb = '0' then
```

**Cause:** The expression `0 as byte` compiles to the VHDL character literal
`'0'` (std_logic), not to `x"00"` (std_logic_vector).  Comparing a `byte`
field (std_logic_vector) to `'0'` (std_logic scalar) is a type mismatch.

**Example failure:**
```livt
var b: byte = data[i]
if (b == 0 as byte) { return i }   // VHDL error: nb = '0'
```

**Workaround:** Cast the byte to int first and compare with integer zero:
```livt
var b: byte = data[i]
var bi: int = b as int
if (bi == 0) { return i }          // works correctly
```

---

## §14 — String Interpolation in `Simulation.Report`

**Status:** Confirmed working (2026-05-18). Verified in `livt-gen-vhdl-verification`.

**Syntax:** Place a local variable name inside `{}` braces inside a string literal passed to `Simulation.Report`:

```livt
var value: int = 42
Simulation.Report("result: {value}")

var count: int = 0
Simulation.Report("result = {result} expected = {expected}")
```

**VHDL emitted:**
- For `int` variables: `report "prefix " & integer'image(var) & " suffix";`
  — produces decimal output.
- For `logic[]` / `byte[]` variables: `report "prefix " & to_string(var) & " suffix";`
  — produces binary/hex representation.

**Multiple variables in one call:** Supported. Each `{name}` segment is concatenated separately.

**Array element indexing:** Supported, e.g. `Simulation.Report("a[0, 0] {a[0, 0]}")`.

**Limitation:** Only local variables and field access are confirmed working. Computed expressions inside `{}` are not confirmed.

---

## §15 — `logic[N] as int` Uses Signed Interpretation

**Status:** Observed (2026-05-18). Likely a compiler bug.

**Symptom:** Casting a `logic[N]` value to `int` uses `to_integer(signed(...))` in VHDL,
producing a signed (two's complement) result:

```livt
var v: logic[8] = 0xFF
var vi: int = v as int   // produces -1, not 255
```

In contrast, `byte as int` uses `to_integer(unsigned(...))` and gives 255.

**Cause:** The code generator applies `to_integer(signed(...))` for all `logic[N]`
casts to `int`. Since `logic[8]` values 0–255 all fit in `int`'s positive range,
the signed interpretation is unexpected.

**Workaround:** Route through `byte` to get unsigned behavior:
```livt
var vi: int = (v as byte) as int   // 255 — unsigned, correct
```

`logic[N] as byte` is a VHDL identity (both are `std_logic_vector`), so no
information is lost, and `byte as int` then uses `to_integer(unsigned(...))` correctly.

---

## §16 — `.Length()` on `string[N]` Variables Generates Invalid VHDL

**Status:** Observed (2026-05-21). Formatter tests avoid this pattern.

**Symptom:** A fixed-size string variable accepts assignment and equality tests,
but calling `.Length()` on it generates VHDL as if the string were a record with
a `length` function member:

```livt
var s: string[3] = "007"
assert s.Length() == 3
```

GHDL reports errors like:

```text
variable "s" does not designate a record
no declaration for "this_..._s_length_in"
```

**What works:** Returning `string[N]` from functions and comparing it with a
literal works in `FormatTest`.

**Workaround:** Treat formatter output lengths as part of the function contract
for now (`ToStringByte` returns `string[3]`, `ToStringInt` returns `string[11]`,
and so on). Do not call `.Length()` on string variables until the compiler
handles it.
