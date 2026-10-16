# `buffer.readvectorf32x*` and `buffer.writevectorf32x*`

**Status**: Open

## Summary

This proposal suggests adding new methods to write & read `vector`s to/from `buffer`s.

## Motivation

The native `vector` type can leverage simd to speed up element-wise operations, making it popular for math.

The `buffer` library provides methods to read and write numeric data types, but in order to store a `vector` to a buffer, code must unpack each element, writing them individually:
```luau
buffer.writef32(theBuffer, offset + 0, theVector.x)
buffer.writef32(theBuffer, offset + 4, theVector.y)
buffer.writef32(theBuffer, offset + 8, theVector.z)
```
Similarly, in order to retrieve vectors from the buffer, code must read each individual float, and then construct a vector from them:
```luau
local theVector = vector.create(
    buffer.readf32(theBuffer, offset + 0),
    buffer.readf32(theBuffer, offset + 4),
    buffer.readf32(theBuffer, offset + 8)
)
```
When `LUA_VECTOR_SIZE` is 4, these patterns extend to a fourth component (`w`).

Each `writef32` or `readf32` performs an individual `memcpy`, and temporarily converts the 32-bit float to a `number` (64-bit double) – one bulk `memcpy` would be more efficient and amenable to simd.

## Design

Adding the following four new methods would fill this performance gap.

```luau
buffer.writevectorf32x2(buf : buffer, offset : number, vec : vector) : ()
buffer.readvectorf32x2(buf : buffer, offset : number) : vector

buffer.writevectorf32x3(buf : buffer, offset : number, vec : vector) : ()
buffer.readvectorf32x3(buf : buffer, offset : number) : vector
```

The `f32` part of the name makes it explicit that these methods operate on 32-bit floats. The methods are defined in terms of vector components rather than the vector's in-memory representation. Each component is converted to an IEEE-754 binary32 value before it is written. On read, each binary32 value is converted to a component while constructing the result vector. Thus, if vectors are internally stored as doubles, writes round each component to binary32 and reads convert the binary32 values to doubles; no vector storage layout is exposed, and the buffer format remains unchanged.

Like all buffer read/write operations, byte order is little-endian. An error is thrown if the read or write would exceed the buffer's bounds.

`buffer.writevectorf32x2(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x` and `vec.y` as two contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y)`

`buffer.readvectorf32x2(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x` and `y` components are determined by reading two contiguous 32-bit floats from `buf` starting at `offset`.
- The resulting vector's `z` component is zero.
- If `LUA_VECTOR_SIZE` is 4, the `w` component of the resulting vector is also zero.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4))`

`buffer.writevectorf32x3(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x`, `vec.y`, and `vec.z` as three contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y); buffer.writef32(buf, offset + 8, vec.z)`

`buffer.readvectorf32x3(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x`, `y`, and `z` components are determined by reading three contiguous 32-bit floats from `buf` starting at `offset`.
- If `LUA_VECTOR_SIZE` is 4, the `w` component of the resulting vector is also zero.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4), buffer.readf32(buf, offset + 8))`

When `LUA_VECTOR_SIZE` is defined to be `4`, two additional methods are defined:

```luau
buffer.writevectorf32x4(buf : buffer, offset : number, vec : vector) : ()
buffer.readvectorf32x4(buf : buffer, offset : number) : vector
```

`buffer.writevectorf32x4(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x`, `vec.y`, `vec.z`, and `vec.w` as four contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y); buffer.writef32(buf, offset + 8, vec.z); buffer.writef32(buf, offset + 12, vec.w)`

`buffer.readvectorf32x4(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x`, `y`, `z`, and `w` components are determined by reading four contiguous 32-bit floats from `buf` starting at `offset`.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4), buffer.readf32(buf, offset + 8), buffer.readf32(buf, offset + 12))`

## Drawbacks

This proposal does not add any brand-new functionality. It increases the API surface of `buffer` by 4 or 6 methods, which could create a maintenance burden. Perhaps improvements to code generation would obviate the need for dedicated vector methods.

## Alternatives

Given there is only one `vector` type, we considered proposing just two methods: `readvector`/`writevector`, that read/write 3 or 4 elements depending on `LUA_VECTOR_SIZE`. But given the existence of 2-element constructors, partial-construction might be popular.

Another alternative is to expose simd operations on `buffer` itself – this might still be a useful extension for non-floating-point operations, but it would result in many more methods.
