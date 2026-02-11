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

Functions for creating/manipulating this type will exist in a new library called 'int64`, which will have the following functions:

`int64.create(n: number) -> int64`

Converts a number to an int64. Will throw an error if the value of the number exceeds the maximum integer, or if there is a fractional component to the number.

`int64.fromstring(str: string, base: number?) -> int64`

Converts a string representation of a number into a int64 in accordance with a specified base or base 10 if no argument is provided.

`int64.tostring(n: int64, b: base?) -> string`

Converts an int64 to a string representation with the given base.

`int64.tonumber(n: int64) -> number`

Converts an int64 to a double, erroring if the value cannot be represented accurately.

`int64.add`, `int64.sub`, `int64.mul`, `int64.div`, `int64.mod`, `int64.udiv`, `int64.umod`, `int64.band`, `int64.bor`, `int64.bnot`, `int64.bxor`, `int64.lt`, `int64.le`, `int64.ult`

Performs the associated operation on a int64 and another int64. Functions prefixed with `u` operate over unsigned values where their normal counterparts assume signedness.
Operators for this type will not be implemented

`int64.lshift`, `int64.rshift`, `int64.arshift`, `int64.lrotate`, `int64.rrotate`, `int64.extract`, `int64.replace`, `int64.btest`, `int64.countrz`, `int64.countlz`

Performs the associated bitwise operation on a int64 by a number.

`buffer.readint64(b: buffer, offset: number) -> int64`

Reads a int64 out of a buffer at the given offset.

`buffer.writeint64(b: buffer, offset: number, value: int64)`

Writes a int64 to a buffer at the given offset.

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
