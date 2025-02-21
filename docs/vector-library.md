# Vector library

**Status**: Implemented

## Summary

Implement a standard library that provides functionality for the vector type.

## Motivation

Currently, vectors are a primitive type implemented internally. All of the heavy work to implement vectors is done, but there is no runtime-agnostic way to use vectors in Luau code. Individual Luau runtimes, such as [Lune](https://github.com/lune-org/lune) or Roblox, must provide their own distinct libraries to work with native vectors which may be inconsistent. In cross-runtime code, this results in performance drawbacks & difficulty utilizing native vectors.

## Design

The default metatable for vectors should now be the `vector` library. While buffers and coroutines do not have this, vectors likely should for ergonomics.

Luau VM can be configured in two different vector value modes.
Default configuration uses vectors with 3 components (`xyz`) and if the `LUA_VECTOR_SIZE` configuration option is set to 4, vector values get an additional fourth `w` component.
This mode will be referred to as '4-wide mode' in this proposal.

### Library functions & constants

This RFC proposes the following basic functions and constants.

`vector.create(x: number, y: number, z: number): vector`

Creates a new vector with the given components. In 4-wide mode, the fourth `w` component value is set to 0.0.

`vector.create(x: number, y: number, z: number, w: number): vector`

Additional constructor available in 4-wide mode. Creates a new vector with the given components.

`vector.magnitude(vec: vector): number`

Calculates the magnitude of a given vector. When in 4-wide mode, this includes the fourth component.

`vector.normalize(vec: vector): vector`

Returns the normalized version (aka unit vector) of a given vector. When in 4-wide mode, this includes the fourth component.

`vector.cross(vec1: vector, vec2: vector): vector`

Returns the cross product of two vectors. If 4-wide vectors are enabled, this function will ignore the fourth component and return the 3-dimensional cross product.

`vector.dot(vec1: vector, vec2: vector): number`

Returns the dot product of two vectors. Uses all components, even in 4-wide mode.

`vector.angle(vec1: vector, vec2: vector, axis: vector?): number`

Returns the angle between two vectors in radians. The axis, if specified, is used to determine the sign of the angle. If 4-wide vectors are enabled, this function will ignore the fourth component and return the 3-dimensional angle.

`vector.floor(vec: vector): vector`

Applies `math.floor` component-wise for each vector. Equivalent of: `vector.create(math.floor(vec.x), etc)`.

`vector.ceil(vec: vector): vector`

Applies `math.ceil` component-wise for each vector. Equivalent of: `vector.create(math.ceil(vec.x), etc)`.

`vector.abs(vec: vector): vector`

Applies `math.abs` component-wise for each vector. Equivalent of: `vector.create(math.abs(vec.x), etc)`.

`vector.sign(vec: vector): vector`

Applies `math.sign` component-wise for each vector. Equivalent of: `vector.create(math.sign(vec.x), etc)`.

`vector.clamp(vec: vector, min: vector, max: vector): vector`

Applies `math.clamp` component-wise for each vector. Equivalent of: `vector.create(math.clamp(vec.x, min.x, max.x), etc)`.

`vector.max(...: vector): vector`

Applies `math.max` component-wise for each vector. Equivalent of: `vector.create(math.max((...).x), etc)`.

`vector.min(...: vector): vector`

Applies `math.min` component-wise for each vector. Equivalent of: `vector.create(math.min((...).x), etc)`.

---

`vector.zero`

Vector where `x=0, y=0, z=0, w?=0`.

`vector.one`

Vector where `x=1, y=1, z=1, w?=1`.

---

### Arithmetic operations

Primitive operators for vectors are already implemented, so this RFC doesn't concern vector arithmetic.

### Component access

Component access is already supported in the VM using the fields 'x', 'y', 'z', 'X', 'Y' and 'Z' (plus 'w' and 'W' in 4-wide mode).
Note that writes to a single component are not supported as the vector value is immutable.
This RFC doesn't define any changes to this behavior.


### Compiler options

Currently, there are 2 compiler options relating to vectors, `vectorLib` & `vectorCtor`. This poses an interesting problem: a builtin library would remove the _requirement_ for such compiler options, however these options still need to be supported. The intuitive solution to this is to leave both compiler options working and maintained, but provide the built-in vector library by default, allowing two vector libraries and two vector constructors.

### Typechecking

A new primitive `vector` type should be added to the type system.

## Future Work

A better vector constructor with new syntax, such as `<x, y, z>` or `(x, y, z)` or `|x, y, z|` or `vector(x, y, z)` or `[x, y, z]` could be implemented. This would make code that does stuff with vectors much less verbose.

The buffer library could include functions to work with vectors, such as `buffer.writevector2`, `buffer.writevector3`, and in 4-wide mode, `buffer.writevector4`. The corresponding read functions would of course be included.

## Drawbacks

As per all additional globals, this creates a new global `vector`. This is a commonly used variable name, and many style guides will advise to avoid using `vector` as a variable name due to the global being added.

Introducing a vector library means introducing more globals. Due to the presence of vectors in performance-intensive code, it can be assumed that the built-in functions will have fastcalls. This means more fastcall slots will be used.

Existing code won't get any improvements from a built-in library. Developers will have to switch over to the built-in library in order to gain any benefit.

## Alternatives

Do nothing; vectors have an internal constructor, which lets the runtime implement vectors, and subsequently a library for vector math.

### Alternative implementations

The function `vector.create` could be skipped entirely in favor of syntax for creating vectors. This should not be done because a function for creating vectors will be nice to have no matter what.

Instead of `vector.magnitude`, the magnitude could be derived from the vector's coordinates themselves, and be accessed by a property instead of a function. There is a downside to this: all current properties of vectors are hard-coded to the VM, so any new property to vectors requires a lot of additional complexity & changes to the VM to allow for this. A library function, however, would be trivial. An easy and quick workaround to the verbosity would be at the runtime/C API level. It's trivial to set the metatable of vectors to be the vector library: this allows for `vec:magnitude()` without much issue.

Instead of ignoring the 4th dimension in `vector.cross`, the function could be disabled when four-dimensional vectors are enabled.
