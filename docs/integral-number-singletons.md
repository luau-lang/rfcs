
# Integral `number` singletons

## Summary
Allow for singletons of the `number` primitive, similar to `boolean` (`true`/`false`).

## Motivation
Programmers may want to use numeric constants as enumerations. Consider:

```luau
type Deserialized = {
    Items: { integer }
}

local function convertFromV1ToV2(_: { string }): { integer }
    -- ...
end

local function deserialize(Serialized): Deserialized
    local deserialized = {}

    if Serialized.Version == 1 then
        deserialized.Items = convertFromV1ToV2(Serialized.Items)
    elseif Serialized.Version == 2 then
        deserialized.Items = Serialized.Items
    else
        error("unexpected version")
    end

    return deserialized
end
```

As it stands, this code cannot be easily typechecked as the type of `Serialized` would have to be narrowed based on the numeric singleton `.Version`.
The desired type of `Serialized` would be:

```luau
type SerializedStructure = {
    Version: 1,
    Items: { string }
} | {
    Version: 2,
    Items: { integer }
}
```

, however, this is currently inexpressible. `Version` would have to be left annotated as `number`, which prevents `deserialize` from typechecking
as the union of `Items` is not narrowed from `{ string } | { integer }`.

Certain runtimes may expose platform-level APIs that use enumerations that are represented as `number`s.

Programmers may want to annotate that a table only has a certain limited set of valid number indices.

## Design

Integral numeric literals valid in the value language (`0`, `0xF2`, `0b11001`, etc) will be made valid for use in the type language, similar to literals
such as `nil` or string literals like `"hello"`. They will be inferred as either their singleton or their supertype (`number`) with the same rules as
string singletons.
Literals with a decimal component are disallowed and raise a syntax error.

Unions of the singletons will be narrowable through refinement similar to strings:

```luau
local x: 1 | 0b10 | 0x3 = getX()

if x == 1 or x == 2 then
    local y: 1 | 2 = x -- No type error
    local z: 3     = x -- Type error
end
```

They will be valid in the table indexer position:

```luau
type T = {
    [1 | 2]: string
}

local t: T = {...}
print(t[1]) -- Ok, 1 <: 1 | 2
print(t[3]) -- Type error, 3 </: 1 | 2
```

Heterogeneous arrays are out of the scope of this RFC.

## Drawbacks
* Increases complexity of the type system

## Alternatives
* Allow for decimals instead of restricting to integers
* Implement singletons for `integer`s
