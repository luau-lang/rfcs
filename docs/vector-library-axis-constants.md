# Vector library axis constants

## Summary

Adds unit axis constants `vector.xaxis`, `vector.yaxis`, and `vector.zaxis` to the vector library, plus `vector.waxis` in 4-wide vector mode.

## Motivation

Unit vectors along the coordinate axes are among the most common vector values in real code. Outside of runtimes with their own vector extensions, developers have to define these manually:

```luau
local X_AXIS = vector.create(1, 0, 0)
local Y_AXIS = vector.create(0, 1, 0)
local Z_AXIS = vector.create(0, 0, 1)
```

Roblox already provides [`Vector3.xAxis`](https://create.roblox.com/docs/reference/engine/datatypes/Vector3#xAxis), [`Vector3.yAxis`](https://create.roblox.com/docs/reference/engine/datatypes/Vector3#yAxis), and [`Vector3.zAxis`](https://create.roblox.com/docs/reference/engine/datatypes/Vector3#zAxis) for this, but those constants are not available in portable Luau code. Adding equivalent builtins to the `vector` library closes that gap.

Alongside this, as of writing the type solver rejects `Vector3` where `vector` is expected and vise versa, even though both are the same primitive at runtime:

```luau
-- TypeError: Expected this to be 'vector', but got 'Vector3'
vector.magnitude(Vector3.yAxis)
```

## Design

The following constants will be added to the `vector` library:

```luau
vector.xaxis = vector.create(1, 0, 0)
vector.yaxis = vector.create(0, 1, 0)
vector.zaxis = vector.create(0, 0, 1)
-- Only exists in 4-wide vector mode
vector.waxis = vector.create(0, 0, 0, 1)
```
