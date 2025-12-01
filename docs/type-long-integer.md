# Long Integer Type

## Summary

A builtin type to represent 64-bit integers.

## Motivation

Current methods of representing 64-bit integers in Luau are done with high-low pairs or by splitting them among vectors which leads to poor ergonomics and may not be sufficient for cases where performance is important.

In the case of a high-low pair implementation each individual number takes 16 bytes (or 24) and needs to be be handled together which can be confusing.

In the case of vector implementations, ergonomics are improved by storing the entire integer in one value however you have the same issues of implementation complexity and lack of integration with the type system.

In both cases, performance is lacking compared to what could be provided by a native implementation.

As Luau grows the restriction to doubles will be an increasing pain point for any use case which involves a 64-bit integer.
System support for 64-bit integers is ubiquitous and their use is widespread.

Specific use cases involve the implementation of specific algorithms (e.g. cryptography, FNV), timestamps, simpler integration with C libraries/runtimes including FFI, larger space for incremental identifiers, wider range of usable bits for libraries that currently are restricted to 2^53-1 

While the above solutions, in addition to runtimes being able to define their own userdata do exist, requiring each runtime to reimplement 64 bit integer support with potentially different semantics could prove damaging to libraries that will have to find a common API between any supported runtime or use their own library to use something as simple as int64.

Representing 64-bit numbers as heap-allocated userdata is sufficient for correctness but adds a layer of indirection for something that can fit within the existing value size. In cases where the use of these numbers is more common, the performance characteristics of different implementations including userdata could be problematic.

Userdata remain the right mechanism for complex or embedder-specific numeric types but 64-bit integers have a sufficiently large problem domain to justify their inclusion as a core value type which may have benefits for performance wider interoperability in the overall ecosystem.

Most other high-level language have builtin support for 64-bit integers, including Lua since 5.3.

## Design

This will be implemented a with new type called "long".

The long type will store a 64 bit integer.

An additional character may be specified at the end of numeric literals `L` which will signify an long integer literal.
Long integer literals will support separators, hex, and binary values.
Long integer literals should be parsed as an alternative to floating point values akin to the following regular expression `\d`.
Long integer literals should support scientific notation declarations e.g. `1e8L` and should produce an error if they result in a fractional part.

Longs will not coerce to a number or string, though type should include a __tostring.

Functions for creating/manipulating this type will exist in a new library called 'long`, which will have the following functions:

`long.tolong(n: number) -> long`

Converts a number to a long. Will throw an error if the value of the number exceeds the maximum integer, or if there is a fractional component to the number.

`long.fromstring(str: string, base: number?) -> long`

Converts a string representation of a number into a long in accordance with a specified base or base 10 if no argument is provided.

`long.tostring(n: long, b: base?) -> string`

Converts a long to a string representation with the given base.

`long.add`, `long.sub`, `long.mul`, `long.div`, `long.mod`, `long.udiv`, `long.umod`, `long.band`, `long.bor`, `long.xor`, `long.lt`, `long.le`, `long.ult`

Performs the associated operation on a long and another long. Functions prefixed with `u` operate over unsigned values where their normal counterparts assume signedness.
Operators for this type will not be implemented

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

Pushes a long value of the specified signed to the stack

`int lua_islong(lua_State *L, int n)`

Determines if a value at index `n` is a long.

`int64_t luaL_checklong(lua_State *L, int index)`

Same behavior as `lua_tolong` and `lua_toulong` except errors if the value is not a long.

`luaopen_long(lua_State *L)`

Registers the `long` library. Included in `luaL_openlibs`

## Drawbacks

This increases the complexity of the VM as it may need to implement separate paths for mathematical operations.

## Alternatives

Do nothing, the workarounds exist and are in use but this will restrict the areas Luau can be used without implementation burden or reliance on an externally maintained implementation of 64-bit integers.
