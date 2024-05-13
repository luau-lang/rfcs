# Vector library

## Summary

Implement a standard library that provides functionality for the vector type.

## Motivation

Currently, vectors are a primitive type implemented internally. All of the heavy work to implement vectors is done, but there is no runtime-agnostic way to use vectors in Luau code. Individual Luau runtimes, such as [Lune](https://github.com/lune-org/lune) or Roblox, must provide their own distinct libraries to work with native vectors which may be inconsistent. In cross-runtime code, this results in performance drawbacks & difficulty utilizing native vectors.

## Design

The default metatable for vectors should now be the `vector` library. While buffers and coroutines do not have this, vectors likely should for ergonomics.

### Library functions & constants

This RFC proposes the following basic functions & constants as a starting point, but as with all builtin libraries, future RFCs can propose additional functions.

---

`vector(x: number?, y: number?, z: number?)`

Creates a vector with 3 components: x, y, z. If the feature flag for wide vectors is enabled, a fourth argument `w: number?` will be introduced.

Due to the common usage of vectors, vector creation should be ergonomic. Therefore, it is probably worth breaking the `create()` naming standard.

`vector.magnitude(vecA: vector): number`

Calculates the magnitude of a given vector.

`vector.unit(vec: vector): vector`

Returns the unit vector (aka normalized vector) of a given vector.

`vector.cross(vecA: vector, vecB: vector): vector`

Returns the cross product of two vectors.

`vector.dot(vecA: vector, vecB: vector): vector`

Returns the dot product of two vectors.

`vector.floor(vec: vector): vector`

Equivalent of `math.floor` for vectors.

`vector.ceil(vec: vector): vector`

Equivalent of `math.ceil` for vectors.

`vector.angle(vecA: vector, vecB: vector): vector`

Returns the angle between two vectors.

`vector.max(input: vector, maximum: vector): vector`

Applies math.max component-wise for each vector. Equivalent of: `vector(math.max(a.x, b.x), etc)`

`vector.min(input: vector, minimum: vector): vector`

Same as `vector.max`. Equivalent of: `vector(math.min(a.x, b.x), etc)`

---

`vector.zero`

Vector where `x=0, y=0, z=0, w?=0`

`vector.one`

Vector where `x=1, y=1, z=1, w?=1`

---

### Arithmetic operations

Primitive operators for vectors are already implemented, so this RFC doesn't concern vector arithmetic.

### Native codegen

In the future, vectors will have special treatment in native codegen. This creates an expectation that the vector library will also have special treatment in codegen, but it isn't clear what that will look like, or if the performance benefits make sense.

### Compiler options

Currently, there are 2 compiler options relating to vectors, `vectorLib` & `vectorCtor`. This poses an interesting problem: a builtin library would remove the _requirement_ for such compiler options, however these options still need to be supported. The intuitive solution to this is to leave both compiler options working and maintained, but provide the built-in vector library by default, allowing two vector libraries and two vector constructors.

### Typechecking

A new primitive `vector` type should be added to the type system.

## Drawbacks

As per all additional globals, this creates a new global `vector`. This is a commonly used variable name, and many style guides will advise to avoid using `vector` as a variable name due to the global being added.

Introducing a vector library means introducing more globals. Due to the prescence of vectors in performance-intensive code, it can be assumed that the built-in functions will have fastcalls. This means more fastcall slots will be used.

Existing code won't get any improvements from a built-in library. Developers will have to switch over to the built-in library in order to gain any benefit.

## Alternatives

Do nothing; vectors have an internal constructor, which lets the runtime implement vectors, and subsequently a library for vector math.

### Alternative implementations

A more standard alternative to `vector(...)` could be something like `vector.create` or `vector.new`. This follows the standard set by `buffer.create`, `coroutine.create`, and `table.create`, and doesn't involve calling a table.

Another alternative to vector creation is special syntax for creating buffers, such as `<x, y, z>`, or `[x, y, z]`. This would require a lot more complexity & discussion, however it would fall more in line with the other primitive types.

Instead of `vector.magnitude`, the magnitude could be derived from the vector's coordinates themselves, and be accessed by a property instead of a function. There is a downside to this: all current properties of vectors are hard-coded to the VM, so any new property to vectors requires a lot of additional complexity & changes to the VM to allow for this. A library function, however, would be trivial. An easy and quick workaround to the verbosity would be at the runtime/C API level. It's trivial to set the metatable of vectors to be the vector library: this allows for `vec:magnitude()` without much issue.
