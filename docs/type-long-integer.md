# 64-bit Integer Type

## Summary

A builtin 'int64' type to represent 64-bit integers.

## Motivation

Current methods of representing 64-bit integers in Luau with default environment are done with high-low pairs or by splitting them among vectors which leads to poor ergonomics and may not be sufficient for cases where performance is important.

In the case of a high-low pair implementation each individual number takes 16 bytes (or 24) and needs to be be handled together which can be confusing.

In the case of vector implementations, ergonomics are improved by storing the entire integer in one value however you have the same issues of implementation complexity and lack of integration with the type system.

In both cases, performance is lacking compared to what could be provided by a native implementation.

As Luau grows the restriction to doubles will be an increasing pain point for any use case which involves a 64-bit integer.
System support for 64-bit integers is ubiquitous and their use is widespread.

While the above solutions, in addition to runtimes being able to define their own userdata do exist, requiring each runtime to reimplement 64 bit integer support with potentially different semantics could prove damaging to libraries that will have to find a common API between any supported runtime or use their own library to use something as simple as int64.

Representing 64-bit numbers as heap-allocated userdata is sufficient for correctness but adds a layer of indirection for something that can fit within the existing value size. In cases where the use of these numbers is more common, the performance characteristics of different implementations including userdata could be problematic.

Userdata remain the right mechanism for complex or embedder-specific numeric types but 64-bit integers have a sufficiently large problem domain to justify their inclusion as a core value type which may have significant benefits for performance as well as wider interoperability in the overall ecosystem.

## Design

This will be implemented a with new type called "int64".

An additional character may be specified at the end of numeric literals `i` which will signify an 64 bit integer literal.
64 bit integer literals will support separators, hex, and binary values.

Operations performing a string formatting of the an int64 should format the number as signed by default.

int64 values will have a built-in equality comparison, but will not have any other operators or metamethods defined.

Functions for creating/manipulating this type will exist in a new library called 'int64`.

### Library

`function int64.create(n: number): int64?`

Converts a number to int64.

If the double number cannot be represented as int64 exactly: has a fractional part, is out of range or is NaN, function returns `nil`.

This behavior is chosen to closely match `math.tointeger` from Lua 5.4 and can have a relatively efficient implementation.

`function int64.fromstring(str: string, base: number?): int64?`

Converts a string representation of a number into a int64.

Number inside the string has to be an integer.
String is allowed to have leading and trailing spaces and number can be preceded by a sign.
If base is not specified, number can have a `0x` or `0X` prefix to be converted in base 16, otherwise base 10 is used.

When base is specified, it has to be in range between 2 and 36 inclusive.
In base 10 and base 16, number is allowed to have a `0x` or `0X` prefix.

If the string doesn't contain a number, `nil` is returned.

This behavior is chosen to match `tonumber`, but excludes floating-point numbers.

`function int64.tostring(n: int64): string`

Converts an int64 to a string representation.

`function int64.tonumber(n: int64): number`

Converts an int64 to a double.

If the value cannot be represented as a double exactly, round to nearest, tie to even rounding mode is used.

`function int64.neg(a: int64): int64`

Negates the value.
Overflow wraps around according to rules of two-complement arithmetic.

`function int64.add(a: int64, b: int64): int64`

Adds `a` to `b`.
Overflow wraps around according to rules of two-complement arithmetic.

`function int64.sub(a: int64, b: int64): int64`

Subtracts `b` from `a`.
Overflow wraps around according to rules of two-complement arithmetic.

`function int64.mul(a: int64, b: int64): int64`

Multiplies `a` and `b`.
Overflow wraps around according to rules of two-complement arithmetic.

`function int64.div(a: int64, b: int64): int64`

Performes signed truncated division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, throws an overflow error.

`function int64.rem(a: int64, b: int64): int64`

Computes remainder of the signed truncated division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, result is 0.

`function int64.idiv(a: int64, b: int64): int64`

Performes signed floored division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, throws an overflow error.

`function int64.mod(a: int64, b: int64): int64`

Performes signed floored modulus division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, result is 0.

`function int64.udiv(a: int64, b: int64): int64`

Performs unsigned division of `a` by `b`.
If `b` is 0, throws an error.

`function int64.urem(a: int64, b: int64): int64`

Computes remainder of the unsigned division of `a` by `b`.
If `b` is 0, throws an error.

`function int64.band(a: int64, b: int64): int64`

Performs a bitwise and of `a` and `b`.

`function int64.bor(a: int64, b: int64): int64`

Performs a bitwise or of `a` and `b`.

`function int64.bnot(a: int64): int64`

Returns a bitwise negation of the input number.

`function int64.bxor(a: int64, b: int64): int64`

Performs a bitwise xor (exclusive or) of `a` and `b`.

`function int64.lt(a: int64, b: int64): boolean`

Compares signed less than (<) comparison of `a` and `b`.

`function int64.le(a: int64, b: int64): boolean`

Compares signed less than or equal (<=) comparison of `a` and `b`.

`function int64.ult(a: int64, b: int64): boolean`

Compares unsigned less than (<) comparison of `a` and `b`.

`function int64.ule(a: int64, b: int64): boolean`

Compares unsigned less than or equal (<=) comparison of `a` and `b`.

`function int64.lshift(n: int64, i: int64): int64`

Shifts `n` to the left by `i` bits (if `i` is negative, a right shift is performed instead). When `i` is outside of `[-63..63]` range, returns 0.

`function int64.rshift(n: int64, i: int64): int64`

Shifts `n` to the right by `i` bits (if `i` is negative, a left shift is performed instead). When `i` is outside of `[-63..63]` range, returns 0.

`function int64.arshift(n: int64, i: int64): int64`

Shifts `n` by `i` bits to the right (if `i` is negative, a left shift is performed instead).
The most significant bit of `n` is propagated during the shift. When `i` is larger than 63, returns an integer with all bits set to the sign bit of `n`. When `i` is smaller than -63, 0 is returned.

`function int64.lrotate(n: int64, i: int64): int64`

Rotates `n` to the left by `i` bits (if `i` is negative, a right rotate is performed instead); the bits that are shifted past the bit width are shifted back from the right.
`i` is interpreted modulo 64.

`function int64.rrotate(n: int64, i: int64): int64`

Rotates `n` to the right by `i` bits (if `i` is negative, a left rotate is performed instead); the bits that are shifted past the bit width are shifted back from the left.
`i` is interpreted modulo 64.

`function int64.extract(n: int64, f: int64, w: int64?): int64`

Extracts bits of `n` at position `f` with a width of `w`.
`w` defaults to 1, so a two-argument version of extract returns the bit value at position `f`.
Bits are indexed starting at 0.
`f` cannot be negative.
`w` has to be positive (> 0).
`f + w` must be less than or equal to 64.

`function int64.replace(n: int64, r: int64, f: int64, w: int64?): int64`

Replaces bits of `n` at position `f` and width `w` with `w` least significant bits of `r`.
`w` defaults to 1, so a three-argument version of replace changes one bit at position `f` to `r` and returns the result.
Bits are indexed starting at 0.
`f` cannot be negative.
`w` has to be positive (> 0).
`f + w` must be less than or equal to 64.

`function int64.btest(a: int64, b: int64): boolean`

Perform a bitwise and `a` and `b` and returns true iff the result is not 0.

`function int64.countrz(n: int64): int64`

Returns the number of consecutive zero bits starting from the right-most (least significant) bit.
Returns 64 if `n` is zero.

`function int64.countlz(n: int64): int64`

Returns the number of consecutive zero bits starting from the left-most (most significant) bit.
Returns 64 if `n` is zero.

`function int64.bswap(n: int64): int64`

Returns `n` with the order of the bytes swapped.

### Extensions to the buffer library

Functions to read and write int64 values are added.
Functions use `int64` in the name as opposed to `i64` to signify the difference of the result/argument type from that of `number`.

`function buffer.readint64(b: buffer, offset: number): int64`

Reads a int64 out of a buffer at the given offset.

`function buffer.writeint64(b: buffer, offset: number, value: int64): ()`

Writes a int64 to a buffer at the given offset.

### Extensions to the string library

`string.format` function is updated to support int64 arguments.

'd', 'i' and '*' format specifiers will format int64 as a signed 64 bit integer number.
'o', 'u', 'x' and 'X' format specifiers will format int64 as an unsigned 64 bit integer number.

### C API

`int64_t lua_toint64(lua_State *L, int index)`

Returns a int64 value on the stack at `index` if no value exists or if it is not a `int64` then returns `0`.

`void lua_pushint64(lua_State *L, int64_t n)`

Pushes a int64 value of the specified signed to the stack

`int lua_isint64(lua_State *L, int n)`

Determines if a value at index `n` is a int64.

`int64_t luaL_checkint64(lua_State *L, int index)`

Same behavior as `lua_toint64` and `lua_touint64` except errors if the value is not a int64.

`luaopen_int64(lua_State *L)`

Registers the `int64` library. Included in `luaL_openlibs`

## Drawbacks

This increases the complexity of the VM as it may need to implement separate paths for mathematical operations.

## Alternatives

Do nothing, the workarounds exist and are in use but this will restrict the areas Luau can be used without implementation burden or reliance on an externally maintained implementation of 64-bit integers.
