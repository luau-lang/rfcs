# N-Dimensional Vectors

## Summary

Update the vector type and library to support vectors with any number of components, using numeric indexing beyond four dimensions.

## Motivation

Luau's `vector` type is currently restricted to 3 or 4 components. While this covers most spatial use cases, development with Luau demands native support for higher-dimensional vectors for reduced GC load and accelerated scalar operations.

Currently, if you need more than 4 dimensions, multiple vectors or tables must be used:

```lua
-- Adding all components of two 6D vectors
local a1, a2 = vector.create(1,2,3), vector.create(4,5,6)
local b1, b2 = vector.create(6,5,4), vector.create(3,2,1)
print(a1 + b1, a2 + b2)

-- Adding all components with tables
local a = {1,2,3,4,5,6}
local b = {6,5,4,3,2,1}

local c = {}
for i = 1, #a do
  c[i] = a[i] + b[i]
end

print(c)
```

This approach has several drawbacks:
- **Extra variables:** Splitting a single logical vector across multiple values increases complexity and makes code harder to manage.
- **Extra bytecode instructions:** Operating across multiple vectors requires repeated passes instead of one, adding unnecessary overhead.
- **Messy code:** Manually grouping and tracking multiple vectors leads to verbose, errorprone, and harder to read code.

## Design

The vector type is extended to support any number of components greater than or equal to 2. The number of components in a vector is fixed at creation time.

### Creating N-Dimensional Vectors
`vector.create` is extended to accept any number of numeric arguments:
```lua
local v3  = vector.create(1, 2, 3)          -- 3D vector (existing behavior)
local v4  = vector.create(1, 2, 3, 4)       -- 4D vector (existing behavior)
local v8  = vector.create(1, 2, 3, 4, 5, 6, 7, 8)  -- 8D vector
local v16 = vector.create(
    0.1, 0.2, 0.3, 0.4,
    0.5, 0.6, 0.7, 0.8,
    0.9, 1.0, 1.1, 1.2,
    1.3, 1.4, 1.5, 1.6
) -- 16D vector
```
At least 2 arguments are required. Passing fewer than 2 arguments is an error.

### Accessing Components
The existing named accessors `.x`, `.y`, `.z`, and `.w` continue to work for the first four components.

Components beyond the 4th are accessed using standard numeric indexing:
```lua
local v = vector.create(1, 2, 3, 4, 5, 6)
print(v.x)   -- 1
print(v.y)   -- 2
print(v[5])  -- 5
print(v[6])  -- 6
```

* Indices 1–4 are equivalent to `.x`, `.y`, `.z`, `.w`, so `v[1] == v.x` is always true.
* Accessing an out-of-bounds index returns `nil`.

### Querying Vector Dimension
A new function, `vector.dim`, returns the number of components in a vector:
`function vector.dim(v: vector): number`
  
```lua
local v8 = vector.create(1, 2, 3, 4, 5, 6, 7, 8)
print(vector.dim(v8))  -- 8

local v3 = vector.create(1, 2, 3)
print(vector.dim(v3))  -- 3
```

## Drawbacks
* `vector.cross` is only meaningful in 3D. Passing a higher-dimensional vector to `vector.cross` will result in a runtime error.
* Libraries that assume all vectors have exactly 3 components will need to use `vector.dim` for validation to support higher-dimensional inputs.

## Alternatives

Do nothing. The multi-vector and table workaround is functional and already in use. However, it has real costs in performance, and readability.
A native solution eliminates these workarounds and allows the runtime to optimize over the full component set in ways impossible with separate vectors or tables.
