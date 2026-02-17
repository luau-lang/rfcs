# 64-bit Integer Type

## Summary

A builtin 'integer' type to represent 64 bit integer numbers.

## Motivation

Current methods of representing 64-bit integers in Luau with the default environment are done with high-low pairs or by splitting them among vectors which leads to poor ergonomics and may not be sufficient for cases where performance is important.

In the case of a high-low pair implementation, each individual number takes 16 bytes (or 24) and needs to be handled together which can be confusing.

In the case of vector implementations, ergonomics are improved by storing the entire integer in one value. However you have the same issues of implementation complexity and lack of integration with the type system.

In both cases, performance is lacking compared to what could be provided by a native implementation.

As Luau grows, the restriction to doubles will be an increasing pain point for any use case which involves a 64-bit integer.
System support for 64-bit integers is ubiquitous and their use is widespread.

Besides the above solutions in userland, runtimes are also able to define their own userdata for 64-bit integers. However, this requires each runtime to reimplement 64-bit integer support themselves, and carries with it the potential of significant differences in their semantics. Such an outcome would harm the ecosystem as a whole, requiring libraries to find a common API across all of their supported runtimes, or otherwise program around differences in their APIs.

Further, representing 64-bit numbers as heap-allocated userdata is sufficient for correctness, but also adds a layer of indirection for something that could fit within the existing value size. In domains where the use of these numbers is more common, the performance characteristics of these varying implementation techniques could be problematic or even prohibitive.

Userdata remains the right mechanism for complex or embedder-specific numeric types, but 64-bit integers have a sufficiently large problem domain to justify their inclusion as a core value type, and bring significant benefits for performance and interoperability in the overall Luau ecosystem.

## Design

This will be implemented with a new type called `integer`.

An additional character may be specified at the end of numeric literals `i` which will signify an 64 bit integer literal.
64 bit integer literals will support separators, hex, and binary values:

```luau
local a = 123i
local b = 1_000i
local c = 0xABABi
local d = 0b1000_1000i
```

Integer literal numbers have to be exactly representable, overflow is a parsing error.

Operations performing a string formatting of an integer should format the number as signed by default.

Integer values will have a built-in equality comparison, but will not have any other operators or metamethods defined.

Integers are never automatically converted to numbers or strings, and vice-versa.
Passing an integer to a function expecting a number (or string) will result in a type error.

While there are no arithmentic operators defined for integers in this RFC, it is still worth noting that operations between `number` and `integer` are also not supported.

Functions for creating and manipulating this type will exist in a new library called 'integer`.

### Library

Unless otherwise specified, all operations interpret integer values as signed two's complement 64 bit numbers.
Operations that treat the value as unsigned are explicitly prefixed with 'u'.

`function integer.create(n: number): integer?`

Converts a number to an integer.

Returns `nil` If the double number cannot be represented as an integer exactly, i.e. it has a fractional part, is out of range or is NaN.

This behavior is chosen to closely match `math.tointeger` from Lua 5.4 and can have a relatively efficient implementation.

`function integer.fromstring(str: string, base: number?): integer?`

Converts a string representation of a number into an integer.

Number inside the string has to be an integer.
String is allowed to have leading and trailing spaces and number can be preceded by a sign.
If base is not specified, number can have a `0x` or `0X` prefix to be converted in base 16, otherwise base 10 is used.

When base is specified, it has to be in range between 2 and 36 inclusive.
In base 10 and base 16, numbers are allowed to have a `0x` or `0X` prefix.

Returns `nil` if the string doesn't contain a number.

This behavior is chosen to match `tonumber`, but excludes floating-point numbers.

`function integer.tostring(n: integer): string`

Converts an integer to a string representation.

`function integer.tonumber(n: integer): number`

Converts an integer to a double.

If the value cannot be represented as a double exactly, round to nearest, tie to even rounding mode is used.

`function integer.neg(a: integer): integer`

Negates the value.
Overflow wraps around according to rules of two-complement arithmetic.

`function integer.add(a: integer, b: integer): integer`

Adds `a` to `b`.
Overflow wraps around according to rules of two-complement arithmetic.

`function integer.sub(a: integer, b: integer): integer`

Subtracts `b` from `a`.
Overflow wraps around according to rules of two-complement arithmetic.

`function integer.mul(a: integer, b: integer): integer`

Multiplies `a` and `b`.
Overflow wraps around according to rules of two-complement arithmetic.

`function integer.div(a: integer, b: integer): integer`

Performs signed truncated division of `a` by `b`.

If `b` is 0, throws a division by zero error.

If `a` is -2^63 and `b` is -1, throws an overflow error.

`function integer.rem(a: integer, b: integer): integer`

Computes remainder of the signed truncated division of `a` by `b`.

If `b` is 0, throws a division by zero error.

If `a` is -2^63 and `b` is -1, result is 0.

`function integer.idiv(a: integer, b: integer): integer`

Performs signed floored division of `a` by `b`.

If `b` is 0, throws a division by zero error.

If `a` is -2^63 and `b` is -1, throws an overflow error.

`function integer.mod(a: integer, b: integer): integer`

Performs signed floored modulus division of `a` by `b`.

If `b` is 0, throws a division by zero error.

If `a` is -2^63 and `b` is -1, result is 0.

`function integer.udiv(a: integer, b: integer): integer`

Performs unsigned division of `a` by `b`.

If `b` is 0, throws an error.

`function integer.urem(a: integer, b: integer): integer`

Computes remainder of the unsigned division of `a` by `b`.

If `b` is 0, throws an error.

`function integer.min(a: integer, b: integer): integer`

Returns the smallest of two integer numbers.

`function integer.max(a: integer, b: integer): integer`

Returns the largest of two integer numbers.

`function integer.clamp(a: integer, min: integer, max: integer): integer`

Returns `a` if the number is in `[min, max]` range; otherwise, returns `min` when `a < min`, and `max` otherwise.

The function errors if `min > max`, consistent with `math.clamp`.

`function integer.band(a: integer, b: integer): integer`

Performs a bitwise and of `a` and `b`.

`function integer.bor(a: integer, b: integer): integer`

Performs a bitwise or of `a` and `b`.

`function integer.bnot(a: integer): integer`

Returns a bitwise negation of the input number.

`function integer.bxor(a: integer, b: integer): integer`

Performs a bitwise xor (exclusive or) of `a` and `b`.

`function integer.lt(a: integer, b: integer): boolean`

Compares signed less than (<) comparison of `a` and `b`.

`function integer.le(a: integer, b: integer): boolean`

Compares signed less than or equal (<=) comparison of `a` and `b`.

`function integer.ult(a: integer, b: integer): boolean`

Compares unsigned less than (<) comparison of `a` and `b`.

`function integer.ule(a: integer, b: integer): boolean`

Compares unsigned less than or equal (<=) comparison of `a` and `b`.

`function integer.lshift(n: integer, i: integer): integer`

Shifts `n` to the left by `i` bits (if `i` is negative, a right shift is performed instead).

When `i` is outside of `[-63..63]` range, returns 0.

`function integer.rshift(n: integer, i: integer): integer`

Shifts `n` to the right by `i` bits (if `i` is negative, a left shift is performed instead).

When `i` is outside of `[-63..63]` range, returns 0.

`function integer.arshift(n: integer, i: integer): integer`

Shifts `n` by `i` bits to the right (if `i` is negative, a left shift is performed instead).

The most significant bit of `n` is propagated during the shift.

When `i` is larger than 63, returns an integer with all bits set to the sign bit of `n`.

When `i` is smaller than -63, 0 is returned.

`function integer.lrotate(n: integer, i: integer): integer`

Rotates `n` to the left by `i` bits (if `i` is negative, a right rotate is performed instead); the bits that are shifted past the bit width are shifted back from the right.

`i` is interpreted modulo 64.

`function integer.rrotate(n: integer, i: integer): integer`

Rotates `n` to the right by `i` bits (if `i` is negative, a left rotate is performed instead); the bits that are shifted past the bit width are shifted back from the left.

`i` is interpreted modulo 64.

`function integer.extract(n: integer, f: integer, w: integer?): integer`

Extracts bits of `n` at position `f` with a width of `w`.

`w` defaults to 1, so a two-argument version of extract returns the bit value at position `f`.

Bits are indexed starting at 0.
`f` cannot be negative.
`w` has to be positive (> 0).
`f + w` must be less than or equal to 64.

`function integer.replace(n: integer, r: integer, f: integer, w: integer?): integer`

Replaces bits of `n` at position `f` and width `w` with `w` least significant bits of `r`.

`w` defaults to 1, so a three-argument version of replace changes one bit at position `f` to `r` and returns the result.

Bits are indexed starting at 0.
`f` cannot be negative.
`w` has to be positive (> 0).
`f + w` must be less than or equal to 64.

`function integer.btest(a: integer, b: integer): boolean`

Perform a bitwise and `a` and `b` and returns true iff the result is not 0.

`function integer.countrz(n: integer): integer`

Returns the number of consecutive zero bits starting from the right-most (least significant) bit.
Returns 64 if `n` is zero.

`function integer.countlz(n: integer): integer`

Returns the number of consecutive zero bits starting from the left-most (most significant) bit.
Returns 64 if `n` is zero.

`function integer.bswap(n: integer): integer`

Returns `n` with the order of the bytes swapped.

### Extensions to the buffer library

Functions to read and write integer values are added.
Functions use `integer` in the name as opposed to `i64` to signify the difference of the result/argument type from that of `number`.

`function buffer.readinteger(b: buffer, offset: number): integer`

Reads an integer out of a buffer at the given offset.

`function buffer.writeinteger(b: buffer, offset: number, value: integer): ()`

Writes an integer to a buffer at the given offset.

### Extensions to the math library

For parity with Lua 5.3+, two additional constants are added to the math library.

`math.maxinteger`

Integer value representing 2^63-1 (`9_223_372_036_854_775_807i`)

`math.mininteger`

Integer value representing -2^63 (`-9_223_372_036_854_775_808i`)

### Extensions to the string library

`string.format` function is updated to support integer arguments.

'd', 'i' and '*' format specifiers will format an integer as a signed 64 bit integer number.

'o', 'u', 'x' and 'X' format specifiers will format an integer as an unsigned 64 bit integer number.

### C API

`int64_t lua_tointeger64(lua_State *L, int idx, int* isinteger)`

Returns a value of an integer at `idx` or 0 on failure.

If `isinteger` is not a null pointer, writes 1 if the value at `idx` was an integer value and 0 on failure.

`void lua_pushinteger64(lua_State *L, int64_t n)`

Pushes an integer value on top of the stack.

`int lua_isinteger64(lua_State *L, int idx)`

Determines if a value at `idx` is an integer.

This function is implemented as a C macro.

`int64_t luaL_checkinteger64(lua_State *L, int narg)`

Returns a value of an integer at `narg` or throws an error on failure.

`int64_t luaL_optinteger64(lua_State *L, int narg, int64_t def)`

Returns a value of an integer at `narg` or `def` if there is no value.

Throws an error if there is a value but it is not an integer.

`luaopen_integer(lua_State *L)`

Registers the `integer` library.
Included in `luaL_openlibs`.

## Drawbacks

This introduces a library with a large set of functions which will likely take a large number of fastcall slots.

The lack of operator overloads causes code with integers to be more verbose, decreasing readability in complex expressions.

Introduction of a new type will require updates to libraries (serialization, input/output) which try to handle all potential Luau values.

The suffix 'i' is standard mathematical notation for imaginary numbers.
Adopting it for integers will likely block proposals for a built-in `complex` number type in the future.

## Alternatives

Do nothing, the workarounds exist and are in use but this will restrict the areas Luau can be used without implementation burden or reliance on an externally maintained implementation of 64-bit integers.

### Operator overloads

This proposal does not introduce overloaded operators aside from equality checks.
This omission is deliberate.

While the lack of operators makes code more verbose, Luau's primary numeric type remains `number`.
The `integer` type is intended to cover specialized use cases, many of which will specifically prefer explicit library functions over operators for performance.

Polymorphic overloaded operators present many challenges and can cause unexpected drops of performance.
This is already true when working with `vector` types without type annotations and given the focus of this proposal, we do not want to provide additional pitfalls.
Monomorphic library functions allow compiler and runtime to optimize more aggressively.

Heavy uses of 64-bit integers often rely on bitwise operations.
Since Luau does not have bitwise operators in its syntax, even with a subset of overloaded operators in place, code will still end up using library functions.

Integer division has multiple conflicting definitions (truncated vs. floored).
Operators like `/` and `%` would force a specific default behavior that might not match the specific rounding that the user intended.
Default behavior might also be less performant, causing developers to still rely on library methods.

Having different library methods allows the developers to choose the specific operation semantics they require.

### Separate signed and unsigned types

It has been suggested to create two separate types for signed and unsigned integers or have it be a property of the value.
Having distinct types would provide the typechecker with the ability to highlight mixed operations and a runtime check could throw an error.

As mentioned above, integers are already not expected to be the default value type to be chosen by developers to work with.
Unsigned integer use is even rarer and other programming languages followed the similar path of only supporting auxiliary operations on them.
Other languages with unsigned integer support like C++ recommend limiting their use in their guidelines.
Given this, we have rejected the idea of distinct signed and unsigned integer types.

Having two distinct integer types increases the API and library surface area and implementation complexity significantly for little gain.

In two's complement representation, a single data type with default signed interpretation covers many signed and unsigned operations in an identical way.
Only a limited subset of operations interpreting data as unsigned is added in this proposal to cover most needs for unsigned integer numbers.

Having library methods instead of operators allows the developer to make an explicit choice of the interpretation required (e.g. `lt` vs `ult`).
Reinterpretation of a negative value as a positive unsigned value can be made deliberately by the choice of function, and is not as error-prone as using an operator.
