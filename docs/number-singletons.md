
# Integral `number` singletons

## Summary
Allow for singletons of the `number` primitive, similar to `boolean` (`true`/`false`) and `string`.

## Motivation
Programmers may want to use numeric constants as enumerations. Consider:

```luau
type Deserialized = {
    items: { string }
}

local function convertFromV1ToV2(_: { number }): { string }
    -- ...
end

local function deserialize(Serialized): Deserialized
    local deserialized = {}

    if Serialized.version == 1 then
        deserialized.items = convertFromV1ToV2(Serialized.data)
    elseif Serialized.version == 2 then
        deserialized.items = Serialized.data
    else
        error("Unexpected version.")
    end

    return deserialized
end
```

As it stands, this code cannot be easily typechecked as the type of `Serialized` would have to be narrowed based on the numeric singleton `.version`.
The desired type of `Serialized` would be:

```luau
type SerializedStructure =
  | {
        version: 1,
        data: { number }
    }
  | {
        version: 2,
        data: { string }
    }
```

, however, this is currently inexpressible. `version` would have to be left annotated as `number`, which prevents `deserialize` from typechecking
as the union of `items` is not narrowed from `{ number } | { string }`.

Certain runtimes may expose platform-level APIs that use enumerations that are represented as `number`s.

Programmers may want to annotate that a table only has a certain limited set of valid number indices.

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

Values within the range of a 64-bit signed integer will be supported (that is, `[-2^63..2^63-1]`).

## Drawbacks
* Increases complexity of the implementation of the type system, and we may begin to infer more complicated
types than we do currently.

## Alternatives
* Allow for decimals
* Intervals/bounds
* Predefined singletons for special floating-point values like NaN or infinity
