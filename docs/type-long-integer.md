
# 64-bit Integer Type

## Summary

Introduce a built-in `integer` type to represent signed 64-bit integer numbers with the ability for users to treat them as unsigned through the standard library.

## Motivation

Luau currently only provides one numeric datatype, `number`, which typically represents a 64-bit floating point number. However, double-precision floats are only capable of safely handling integer values up to &plusmn;2^53 before losing numerical precision, which is not desirable in some cases.

While it is possible for users to emulate 64-bit integers in Luau with high-low pairs and vectors, these solutions yield poor ergonomics and performance. Runtimes may also define their own custom userdata implementations for 64-bit integers, however this requires the embedder to reimplement 64-bit integer support themselves and also carries a heavy API and performance cost.

Introducing a native 64-bit integer VM type would alleviate the need for users to emulate their own 64-bit integer datatypes and for embedders to define their own. Since 64-bit integers have a sufficiently large problem domain, their inclusion as a core language feature is justified.

## Design

A new value type named `integer` will be introduced. This type represents a signed 64-bit integer by default that can also be treated as unsigned using the standard library.

The character `i` may be specified at the end of numeric literals to signify that it is a signed 64-bit integer literal. While unsigned 64-bit integer literals are not currently defined in this RFC, future RFCs are free to introduce them with the `u` suffix.

```luau
local a = 123i
local b = 1_000i
local c = 0xABABi
local d = 0b1000_1000i
```

Integer literal numbers must be exactly representable, with overflow causing a parsing error.

```luau
-- these literals are illegal
local a = 1.23i
local b = 9_223_372_036_854_775_808i
```

Operations performing string formatting should format the integer as signed by default.

### `tonumber`

The `tonumber` global will accept integers and convert them to numbers, treating the integer as signed by default. If the value cannot be represented as a `number` exactly, it will attempt to round the result to the nearest whole number representable.

### Metamethods

`__tostring`, arithmetic and comparison metamethods will also be supported and will treat the integer as signed by default.

#### `function __tostring(n: integer): string`

Converts the integer into a decimal string, treating the integer as signed.

#### `function __unm(n: integer): integer`

Negates the value, treating the integer as signed.
Overflow wraps around according to the rules of two's-complement arithmetic.

#### `function __add(a: integer, b: integer): integer`

Adds `a` to `b`. Functions identical for signed and unsigned operands.
Overflow wraps around according to the rules of two's-complement arithmetic.

#### `function __sub(a: integer, b: integer): integer`

Subtracts `b` from `a`. Functions identical for signed and unsigned operands.
Overflow wraps around according to the rules of two's-complement arithmetic.

#### `function __mul(a: integer, b: integer): integer`

Multiplies `a` with `b`. Functions identical for signed and unsigned operands.
Overflow wraps around according to the rules of two's-complement arithmetic.

#### `function __div(a: integer, b: integer): integer`

Performs signed truncated division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, throws an overflow error.

#### `function __idiv(a: integer, b: integer): integer`

Performs signed floored division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, throws an overflow error.

#### `function __mod(a: integer, b: integer): integer`

Performs signed floored modulus division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, returns 0.

#### `function __eq(a: integer, b: integer): boolean`
Compares the equality between both integers, agnostic to signed-ness.

#### `function __lt(a: integer, b: integer): boolean`
Compares signed less-than (<) between `a` and `b`.

#### `function __le(a: integer, b: integer): boolean`
Compares signed less-than-or-equal (<=) between `a` and `b`.


### Library

Functions for creating and manipulating the integer type will exist in a new library called `integer`.

#### `function integer.create(n: number): integer?`
Converts a number to a signed integer. Returns `nil` if the number cannot be represented exactly as an integer, i.e. it has a fractional part, is out of range, or is NaN.

This behavior is chosen to closely match `math.tointeger` from Lua 5.4 and can have a relatively efficient implementation.

#### `function integer.create(s: string): integer?`
Converts a string representation of a number into an integer, choosing which signed-ness to parse by based on the range of the integer represented.

The number represented inside the string must be exactly representable as a signed or unsigned integer. Follows the same semantics as `tonumber` in relation to formatting, i.e. the string is allowed to have leading and trailing spaces, use `0x` prefix and `pN` suffix for hexadecimal, etc.

Like `tonumber`, the base has a range between 2 and 36 inclusive.
If the string cannot be parsed into a valid signed or unsigned integer, returns `nil`. This behavior is chosen to match `tonumber`.

#### `function integer.tostringunsigned(n: integer): string`
Converts the integer to a string, treating it as unsigned. For signed, use the default `tostring`.

#### `function integer.tonumberunsigned(n: integer): number`
Converts the integer to a number, treating it as unsigned. For signed, use the default `tonumber`.
Just like `tonumber`, if the value cannot be represented exactly, the result is truncated and rounded to the nearest whole number.

#### `function integer.udiv(a: integer, b: integer): integer`
Performs unsigned division of `a` by `b`.
If `b` is 0, throws an error.

#### `function integer.rem(a: integer, b: integer): integer`
Computes the remainder of signed truncated division of `a` by `b`.
If `b` is 0, throws a division by zero error.
If `a` is -2^63 and `b` is -1, the result is 0.

#### `function integer.urem(a: integer, b: integer): integer`
Computes the remainder of the unsigned division of `a` by `b`.
If `b` is 0, throws an error.

#### `function integer.ult(a: integer, b: integer): boolean`
Compares unsigned less-than (<) between `a` and `b`.

#### `function integer.ule(a: integer, b: integer): boolean`
Compares unsigned less-than-or-equal (<=) between `a` and `b`.

#### `function integer.band(x: integer, ...: integer): integer`
Performs the bitwise-and between the arguments.

Mirrors `bit32.band`.

#### `function integer.bor(x: integer, ...: integer): integer`
Performs the bitwise-or between the arguments.

Mirrors `bit32.bor`.

#### `function integer.bxor(x: integer, ...: integer): integer`
Performs the bitwise-xor (exclusive or) between the arguments.

Mirrors `bit32.bxor`.

#### `function integer.bnot(x: integer): integer`
Returns the bitwise negation of the input number.

Mirrors `bit32.bnot`.

#### `function integer.lshift(n: integer, i: integer): integer`
Shifts `n` to the left by `i` bits (if `i` is negative, a right shift is performed instead.)
When `i` is outside of `[-63..63]` range, returns 0.

Mirrors `bit32.lshift`.

#### `function integer.rshift(n: integer, i: integer): integer`
Shifts `n` to the right by `i` bits (if `i` is negative, a left shift is performed instead.)
When `i` is outside of `[-63..63]` range, returns 0.

Mirrors `bit32.rshift`.

#### `function integer.arshift(n: integer, i: integer): integer`
Shifts `n` by `i` bits to the right (if `i` is negative, a left shift is performed instead.)
The most significant bit of `n` is propagated during the shift.

When `i` is larger than 63, returns an integer with all bits set to the sign bit of `n`.
When `i` is smaller than -63, returns 0.

Mirrors `bit32.arshift`.

#### `function integer.lrotate(n: integer, i: integer): integer`
Rotates `n` to the left by `i` bits (if `i` is negative, a right rotate is performed instead.) The bits that are shifted past the bit width are shifted back from the right.
`i` is interpreted modulo 64.

Mirrors `bit32.lrotate`.

#### `function integer.rrotate(n: integer, i: integer): integer`
Rotates `n` to the right by `i` bits (if `i` is negative, a left rotate is performed instead.) The bits that are shifted past the bit width are shifted back from the left.
`i` is interpreted modulo 64.

Mirrors `bit32.rrotate`.

#### `function integer.extract(n: integer, f: integer, w: integer?): integer`
Extracts bits of `n` at position `f` with a width of `w`.
`w` defaults to 1, so a two-argument version of extract returns the bit value at position `f`.

Bits are indexed starting at 0. `f` cannot be negative. `w` has to be positive (> 0.) `f + w` must be less than or equal to 64.

Mirrors `bit32.extract`.

#### `function integer.replace(n: integer, r: integer, f: integer, w: integer?): integer`
Replaces bits of `n` at position `f` and width `w` with `w` least significant bits of `r`.
`w` defaults to 1, so a three-argument version of replace changes one bit at position `f` to `r` and returns the result.

Bits are indexed starting at 0. `f` cannot be negative. `w` has to be positive (> 0.) `f + w` must be less than or equal to 64.

Mirrors `bit32.replace`.

#### `function integer.btest(integers: integer, ...: integer): boolean`
Performs a bitwise-and of the integers supplied and returns if the result isn't 0.

Mirrors `bit32.btest`.

#### `function integer.countrz(n: integer): integer`
Returns the number of consecutive zero bits starting from the right-most (least significant) bit.
Returns 64 if `n` is zero.

Mirrors `bit32.countrz`.

#### `function integer.countlz(n: integer): integer`
Returns the number of consecutive zero bits starting from the left-most (most significant) bit.
Returns 64 if `n` is zero.

Mirrors `bit32.countlz`.

#### `function integer.bswap(n: integer): integer`
Returns `n` with the order of the bytes swapped.

Mirrors `bit32.bswap`.

### Extensions to the buffer library

Functions to read and write integer values will be added to the buffer library.
Functions use `integer` in the name as opposed to `i64` to signify the difference of the result/argument type from that of `number`.

#### `function buffer.readinteger(b: buffer, offset: number): integer`
Reads an integer out of a buffer at the given offset.

#### `function buffer.writeinteger(b: buffer, offset: number, value: integer): ()`
Writes an integer to a buffer at the given offset.

### Extensions to the math library

For parity with Lua 5.3+, two additional constants will be added to the math library.

#### `math.maxinteger`
Integer value representing the maximum signed 64-bit integer (2^63-1 or `9_223_372_036_854_775_807i`.)

#### `math.mininteger`
Integer value representing the minimum signed 64-bit integer (-2^63 or `-9_223_372_036_854_775_808i`.)

### Extensions to the string library

The function `string.format` will be updated to support integer arguments.

The `%d`, `%i` and `%*` format specifiers will format integers as signed 64-bit integer numbers.
The `%o`, `%u`, `%x` and `%X` format specifiers will format integers as unsigned 64-bit integer numbers.

### C API

#### `int64_t lua_tointeger64(lua_State *L, int idx, int* isinteger)`
Returns a value of an integer at `idx` or 0 on failure.
If `isinteger` is not a null pointer, writes 1 if the value at `idx` was an integer value and 0 on failure.

#### `void lua_pushinteger64(lua_State *L, int64_t n)`
Pushes a integer value on top of the stack.

#### `int lua_isinteger64(lua_State *L, int idx)`
Determines if a value at `idx` is an integer.
This function is implemented as a C macro.

#### `int64_t luaL_checkinteger64(lua_State *L, int narg)`
Returns a value of an integer at `narg` or throws an error on failure.

#### `int64_t luaL_optinteger64(lua_State *L, int narg, int64_t def)`
Returns a value of an integer at `narg` or `def` if there is no value.
Throws an error if there is a value but it is not an integer.

#### `luaopen_integer(lua_State *L)`
Registers the `integer` library.
Included in `luaL_openlibs`.

## Drawbacks

This increases the complexity of the VM as it may need to implement separate paths for mathematical operations.

The `integer` library implements many functions for parity with the `bit32`, which creates a precedent for future `bit32` additions to also be introduced to the `integer` library.

## Alternatives

As always, do nothing. The workarounds exist and are in use, but this will restrict the areas Luau can be used in without implementation burden or reliance on an externally maintained implementation of 64-bit integers.

Supporting integers in `tonumber` may be controversial and could be scrapped in favor of a library function instead. However, supporting `tonumber` makes the most logical sense and provides a clear avenue for converting signed integers to numbers.

The offset arguments for bitwise functions such as `integer.lshift` and `integer.replace` currently only support integers but may be extended to also support typical numbers for ergonomics. This is left out of the RFC for simplicity but can be reintroduced in a later RFC.

An earlier version of this RFC proposed adding utility functions such as `integer.min` and `integer.clamp`. This was scrapped as it implies duplicated functions for signed/unsigned variants. Future RFCs can explore this area.

An earlier version of this RFC also did not include any metamethods for integers, supplying signed arithmetic and comparison operations in the standard library. This was scrapped as not supporting operators for a numeric datatype seems wrong and providing the signed/sign-agnostic variants as operators while leaving the unsigned variants in greatly reduces the size of the standard.
