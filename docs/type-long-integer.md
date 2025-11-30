# Long Integer Type

## Summary

A builtin type to represent 64-bit integers.

## Motivation

Current methods of representing 64-bit integers in Luau are done with high-low pairs or by splitting them among vectors which leads to poor ergonomics and may not be sufficient for cases where performance is important.

In the case of a high-low pair implementation each individual number takes 16 bytes (or 24) and needs to be be handled together which can be confusing.

In the case of vector implementations, ergonomics are improved by storing the entire integer in one value however you have the same issues of implementation complexity and lack of integration with the type system.

In both cases, performance is lacking compared to what could be provided by a native implementation.

As Luau grows the restriction to doubles will be an increasing pain point for any use case which involves a 64-bit integer.
System support for 64-bit integers is ubiquitous their use is widespread.

While a runtime could implement 64-bit integers with userdata, performing an allocation for each integer (which can easily fit within an existing Luau value) is an onerous requirement.
A lightuserdata could avoid this allocation this but implementors will not be able to use operators or have any integration with the type system.

Most other high-level language have builtin support for 64-bit integers, including Lua since 5.3.

## Design

This will be implemented a with new type called "long".

The `long` type does not posess any means to differentiate signedness and unsignedness, this is left to the API.

An additional character may be specified at the end of numeric literals `L` which will signify an long integer literal.
Long integer literals will support separators, hex, and binary values.
Long integer literals should be parsed as an alternative to floating point values akin to the following regular expression `\d`.
Long integer literals should support scientific notation declarations `1e8` and should produce an error if they result in a fractional part.

Longs will not have any conversions.

Functions for creating/manipulating this type will exist in a new library called 'long`, which will have the following functions:

`long.fromstring(str: string, base: number?) -> long`

Converts a string representation of a number into a long in accordance with a specified base or base 10 if no argument is provided.

`long.tostring(n: long, signedness: boolean) -> string`

Converts a long to a string representation of the long in accordance with the specified signedness and base or base 10 if no argument is provided.

`long.add`, `long.sub`, `long.mul`, `long.div`, `long.udiv`, `long.mod`, `long.umod`, `long.band`, `long.bor`, `long.xor`, `long.ult`, `long.lt`, `long.le`

Performs the associated operation on a long and another long.
Operators `==`, `~=`, `+`, `-`, `*` should be implemented, erroring on use with numbers like any other type that has no valid conversion.

`long.lshift`, `long.rshift`, `long.arshift`

Performs the associated bitwise operation on a long by a number.

`buffer.readlong(b: buffer, offset: number) -> long`

Reads a long out of a buffer at the given offset.

`buffer.writelong(b: buffer, offset: number, value: long)`

Writes a long to a buffer at the given offset.

### C API

`int64_t lua_tolong(lua_State *L, int index)`

Returns a long value on the stack at `index` if no value exists or if it is not a `long` then returns `0`.

`void lua_pushlong(lua_State *L, int64_t n)`

Pushes a long value to the stack

`lua_islong(L, n)`

Helper macro defined to determine if a value at `n` is a long.

`int64_t luaL_checklong(lua_State *L, int index)`

Same behavior as `lua_tolong` except errors if the value is not a long.

`luaopen_long(lua_State *L)`

Registers the `long` library. Included in `luaL_openlibs`

## Drawbacks

The lack of an way to determine signedness adds implementation complexity requiring duplicate methods for some signedness-sensitive operations (div, mod, less than).

This increases the complexity of the VM as it may need to implement separate path for specific mathematical operations.

## Alternatives

Do nothing, the workarounds exist and are in use but this will restrict the areas Luau can be used without implementation burden or reliance on an externally maintained implementation of 64-bit integers.
