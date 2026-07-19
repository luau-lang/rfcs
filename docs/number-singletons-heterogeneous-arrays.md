
# Integral `number` singletons and heterogeneous arrays

## Summary
Allow for singletons of the `number` primitive, similar to `boolean` (`true`/`false`).

Allow for them to be used like table properties to form heterogeneous arrays.

## Motivation
Programmers may want to use numeric constants as enumerations. Consider:

```luau
type Deserialized = {
    Items: { string }
}

local function convertFromV1ToV2(_: buffer): { string }
    -- ...
end

local function deserialize(Serialized): Deserialized
    local deserialized = {}

    if Serialized.Version == 1 then
        deserialized.Items = convertFromV1ToV2(Serialized.Data)
    elseif Serialized.Version == 2 then
        deserialized.Items = Serialized.Data
    else
        error("unexpected version")
    end

    return deserialized
end
```

As it stands, this code cannot be easily typechecked as the type of `Serialized` would have to be narrowed based on the numeric singleton `.Version`.
The desired type of `Serialized` would be:

```luau
type SerializedStructure =
  | {
        Version: 1,
        Data: { string }
    }
  | {
        Version: 2,
        Data: buffer
    }
```

, however, this is currently inexpressible. `Version` would have to be left annotated as `number`, which prevents `deserialize` from typechecking
as the union of `Items` is not narrowed from `{ string } | buffer`.

Certain runtimes may expose platform-level APIs that use enumerations that are represented as `number`s.

Programmers may want to annotate that a table only has a certain limited set of valid number indices.

They may want to annotate that elements at different indices of a table have different types for their corresponding values (heterogeneous arrays).

## Design

Integral numeric literals valid in the value language (`0`, `0xF2`, `0b11001`, etc) will be made valid for use in the type language, similar to literals
such as `nil` or string literals like `"hello"`. They will be inferred as either their singleton or their supertype (`number`) with the same standard bidirectional inference rules as
string singletons.
Literals with a decimal component are unsupported syntax.

Unions of the singletons will be narrowable through refinement similar to strings:

```luau
local x: 1 | 0b10 | 0x3 = getX()

if x == 1 or x == 2 then
    local y: 1 | 2 = x -- No type error
    local z: 3     = x -- Type error
end
```

They may be used to define a dependent type similar to table properties:

```luau
type T = {
    [1]: string,
    [2]: vector
}

local x: T = { "hello", vector.zero }
local y = x[1] --> string
local z = x[2] --> vector

local a: 1 | 2 = getIndex()
local w = x[a] --> string | vector
```

They intersect like table properties:

```luau
type T = {
    [1]: boolean
}

type U = {
    [2]: buffer
}

type V = T & U --> { [1]: boolean, [2]: buffer }
```

## Drawbacks
* Increases complexity of the type system

## Alternatives
* Allow for decimals
* Singletons for `integer`s
* Intervals/bounds
* Predefined singletons for special floating-point values like NaN or infinity
