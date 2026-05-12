# `buffer.writevector*` and `buffer.readvector*`

**Status**: Open

## Summary

This proposal suggests adding new methods to write & read `vector`s to/from `buffer`s, and a `vector.width` field for querying the environment's vector width.

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

### `vector.width`

`LUA_VECTOR_SIZE` is a compile-time option on the Luau VM that determines whether the native `vector` type has 3 or 4 components. Luau does not currently expose the configured value to Luau code, which makes it difficult to write portable code that behaves correctly under both configurations (e.g. deciding how many floats to read back from a buffer).

This proposal adds a new `vector.width` field:

```luau
vector.width : number
```

- Evaluates to `3` when `LUA_VECTOR_SIZE` is `3`, and `4` when `LUA_VECTOR_SIZE` is `4`.
- Exposed as a field (not a function) because the value is constant for the lifetime of the VM, mirroring how `math.pi` and `math.huge` are exposed.

### `buffer.writevector*` and `buffer.readvector*`

Adding the following four new methods would fill the performance gap described above.

```luau
buffer.writevector2(buf : buffer, offset : number, vec : vector) : ()
buffer.readvector2(buf : buffer, offset : number) : vector

buffer.writevector3(buf : buffer, offset : number, vec : vector) : ()
buffer.readvector3(buf : buffer, offset : number) : vector
```

Like all buffer read/write operations, byte order is little-endian. An error is thrown if the read or write would exceed the buffer's bounds.

`buffer.writevector2(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x` and `vec.y` as two contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y)`

`buffer.readvector2(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x` and `y` components are determined by reading two contiguous 32-bit floats from `buf` starting at `offset`.
- The resulting vector's `z` component is zero.
- If `LUA_VECTOR_SIZE` is 4, the `w` component of the resulting vector is also zero.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4))`

`buffer.writevector3(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x`, `vec.y`, and `vec.z` as three contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y); buffer.writef32(buf, offset + 8, vec.z)`

`buffer.readvector3(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x`, `y`, and `z` components are determined by reading three contiguous 32-bit floats from `buf` starting at `offset`.
- If `LUA_VECTOR_SIZE` is 4, the `w` component of the resulting vector is also zero.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4), buffer.readf32(buf, offset + 8))`

When `LUA_VECTOR_SIZE` is defined to be `4`, two additional methods are defined:

```luau
buffer.writevector4(buf : buffer, offset : number, vec : vector) : ()
buffer.readvector4(buf : buffer, offset : number) : vector
```

`buffer.writevector4(buf : buffer, offset : number, vec : vector) : ()`
- Writes `vec.x`, `vec.y`, `vec.z`, and `vec.w` as four contiguous 32-bit floats into `buf`, starting at `offset`.
- equivalent to `buffer.writef32(buf, offset, vec.x); buffer.writef32(buf, offset + 4, vec.y); buffer.writef32(buf, offset + 8, vec.z); buffer.writef32(buf, offset + 12, vec.w)`

`buffer.readvector4(buf : buffer, offset : number) : vector`
- Constructs a new `vector`, whose `x`, `y`, `z`, and `w` components are determined by reading four contiguous 32-bit floats from `buf` starting at `offset`.
- equivalent to `vector.create(buffer.readf32(buf, offset), buffer.readf32(buf, offset + 4), buffer.readf32(buf, offset + 8), buffer.readf32(buf, offset + 12))`

## Drawbacks

This proposal does not add any brand-new functionality. It increases the API surface of `buffer` by 4 or 6 methods, which could create a maintenance burden. Perhaps improvements to code generation would obviate the need for dedicated vector methods.

## Alternatives

Given there is only one `vector` type, we considered proposing just two methods: `readvector`/`writevector`, that read/write 3 or 4 elements depending on `LUA_VECTOR_SIZE`. With `vector.width` exposed, such methods would be usable portably – a reader could advance its cursor by `vector.width * 4` bytes. However, this only works when the reading and writing VMs share the same `LUA_VECTOR_SIZE`; serializing across environments with different widths would still require the width to be recorded out-of-band. The explicit `readvector2`/`readvector3`/`readvector4` spelling avoids this ambiguity and preserves partial-construction, which we expect to be popular given the existence of 2-element constructors.

For `vector.width`, we considered a function (`vector.width()`) or a boolean (`vector.isvector4`) instead of a field. A field is preferred because the value is constant for the VM's lifetime and because it generalizes cleanly if a future `LUA_VECTOR_SIZE` value is ever added.

Another alternative is to expose simd operations on `buffer` itself – this might still be a useful extension for non-floating-point operations, but it would result in many more methods.
