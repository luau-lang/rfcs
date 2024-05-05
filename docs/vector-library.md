# Vector library

## Summary

Implement a standard library that provides functionality for the vector type.

## Motivation

Currently, vectors are a primitive type implemented internally. All of the heavy work to implement vectors is done, but there is no runtime-agnostic way to use vectors in Luau code. This creates a struggle to use native vectors; a Luau runtime [Lune](https://github.com/lune-org/lune) doesn't currently use native vectors. This results in performance drawbacks & difficulty utilizing vectors in cross-runtime code.

## Design

Implement a standard library for creating & using the existing vector type.

### Library functions

It's important to keep in mind that this list of implementable functions isn't intended to be exhaustive, but rather to serve as a starting point.

---

`vector(x: number?, y: number?, z: number?)`

Creates a vector with 3 components: x, y, z. If the feature flag for wide vectors is enabled, a fourth argument `w: number?` will be introduced. As per standard, vectors wouldn't have a metatable by default. A vector's metatable would need to be set by the C API `lua_setmetatable`.

Due to the common usage of vectors, vector creation should be ergonomic. Therefore, it is probably worth breaking the `create()` naming standard.

`vector.magnitude(vecA: vector): vector`

Calculates the magnitude of a given vector.

`vector.unit(vec: vector): vector`

Returns the unit vector (aka normalized vector) of a given vector.

`vector.cross(vecA: vector, vecB: vector): vector`

Returns the cross product of two vectors.

`vector.dot(vecA: vector): vector`

Returns the dot product of a vector.

---

### Arithmetic operations

Primitive operators for vectors are already implemented, so this RFC doesn't concern vector arithmetic.

### Native codegen

In the future, vectors will have special treatment in native codegen. This creates an expectation that the vector library will also have special treatment in codegen, but it isn't clear what that will look like, or if the performance benefits make sense.

### Compiler options

Currently, there are 2 compiler options relating to vectors, `vectorLib` & `vectorCtor`. This poses an interesting problem: a builtin library would remove the _requirement_ for such compiler options, however these options still need to be supported. The intuitive solution to this is to leave both compiler options working and maintained, but provide the built-in vector library by default, allowing two vector libraries and two vector constructors.

## Drawbacks

As per all additional globals, this creates a new global `vector`. This is a commonly used variable name, and many style guides will advise to avoid using `vector` as a variable name due to the global being added.

Introducing a vector library means introducing more globals. Due to the prescence of vectors in performance-intensive code, it can be assumed that the built-in functions will have fastcalls. This means more fastcall slots will be used.

Existing code won't get any improvements from a built-in library. Developers will have to switch over to the built-in library in order to gain any benefit.

## Alternatives

Do nothing; vectors have an internal constructor, which lets the runtime implement vectors, and subsequently a library for vector math.

### Alternative implementations

A more standard alternative to `vector(...)` could be something like `vector.create` or `vector.new`. This follows the standard set by `buffer.create`, `coroutine.create`, and `table.create`, and doesn't involve calling a table.

Instead of `vector.magnitude`, the magnitude could be derived from the vector's coordinates themselves, and be accessed by a property instead of a function. There is a downside to this: all current properties of vectors are hard-coded to the VM, so any new property to vectors requires a lot of additional complexity & changes to the VM to allow for this. A library function, however, would be trivial. An easy and quick workaround to the verbosity would be at the runtime/C API level. It's trivial to set the metatable of vectors to be the vector library: this allows for `vec:magnitude()` without much issue.
