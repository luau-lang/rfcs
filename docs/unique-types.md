# Unique Types
---
## Summary
---
This RFC proposes adding support for unique types to luau.

Unique types are an extension to the type checker, and make no changes to runtime semantics.

## Motivation
---
Since Luau uses structural typing, there is no way to make a primitive distinct. If there are two types PlayerId and AssetId and they are both strings, the type checker allows a PlayerId to be passed into a function expecting AssetId because they are both just string.

Current workarounds like tagging (string & { _tag: "PlayerId" }) are messy and confuse autocomplete.

Unique types solve this by being able to be composed with existing types to make them completely unique, kind of like a tag.
This relies on the behavior of intersections in that an intersection T can only be cast into another intersection U if T is the same/is a subtype of U.
A structure that is an intersection between some type T and a unique type U will not be able to be cast into another structure that is an intersection between the same type T and another unique type V.

## Design
---
The proposed syntax to create a unique type is to define it using `unique type TypeName`, it does not have an = symbol after it because all unique types are are opaque unique types, they do not contain any additional information beyond that.

The name `unique` was chosen as it clearly conveys that this specific type is unique and is not the same as any other type.
Unique types of the same name defined in different files will, of course, still be unique as their own "primitive" type, and cannot be cast to eachother.

### Behavior with intersections

Since unique types simply act as a unique opaque type, this means intersecting them is quite trivial and isn't much different from intersecting other primitive types such as `unknown`.

### Behavior with literals

When assigning a literal value to a variable, a cast will not be implicitly performed. It is expected that unique types will almost exclusively be generated within APIs, rather than written as literals.
Illustrated in code:

```luau
unique type _useridtag
type UserId = _useridtag & number

local user1: UserId = 1  -- doesnt work, error
local user2: UserId = 2 :: UserId  -- works!
```

It's important to note that the only reason this works is because unique types are opaque, and should work similarly to `unknown` wherein it'll "inherit" anything it's composed with

### Casting semantics
Unique types can be casted to other unique types, or other structural types provided the types are compatible in a structural manner. That is to say:

```luau
unique type Vector
  type Vec2 = Vector & { x: number, y: number }
  type Vec3 = Vector & { x: number, y: number, z: number }

local vec2_1 = { x=1, y=1 } :: Vec2
local vec3_1 = { x=1, y=1, z=2 } :: Vec3

local vec2_2: Vec2 = vec3_1 :: Vec2  -- Works, "x" and "y" are present, which is all that's required
local vec3_2: Vec3 = vec2_1 :: Vec3  -- Doesnt work, "z" is missing from the type
```

Again, it's important to note that the only reason this works is because unique types are opaque, and should work similarly to `unknown` wherein it'll "inherit" anything it's composed with

### Enum-like behavior

Because unique types are used as tags within intersections, this means you can get behavior similar to enums

```luau
unique type SomethingType
  type Item = SomethingType & "Item"
  type Weapon = SomethingType & "Weapon"

local function getSomething(ty: SomethingType)

getSomething("Weapon") -- type error: expected SomethingType, got "Weapon"
getSomething("Weapon" :: Item) -- errors, could not convert "Weapon" into "Item"
getSomething("Item" :: Item) -- works!
getSomething("Weapon" :: Weapon) -- works!
```

### Mutually inclusive types

A unique type can be composed within multiple type definitions to create mutually inclusive types which can only be cast to eachother or the tag but nothing else

```luau
unique type _coordtag
  type CoordX = _coordtag & number
  type CoordY = _coordtag & number
  type CoordZ = _coordtag & number

-- These can all be converted to each other
local function swapAxes(x: CoordX, y: CoordY): (CoordY, CoordX)
    return x :: CoordY, y :: CoordX
end

local function promoteToZ(x: CoordX): CoordZ
    return x :: CoordZ
end

local x: CoordX = 10 :: CoordX
local y: CoordY = 20 :: CoordY

local newY, newX = swapAxes(x, y)  -- Valid
local z: CoordZ = promoteToZ(x)    -- Valid

-- But they're still isolated from regular numbers
local regular: number = x  -- error: CoordX is not assignable to number

-- And from other unique type families
unique type _othertag
type OtherId = _othertag & number
local other: OtherId = x  -- error: Different unique tags
```

### Operations on unique types

Since unique types themselves act like `unknown`, this means any sort of operations on a unique type itself will error and is invalid.
However, if you compose a unique type, it'll have all the operations of the types you're composing it with.

```luau
unique type T
type U = T & string

local t: T
local u: U

print(t .. "hi") -- doesnt work, t doesnt have __concat
print(u .. "hi") -- Works, type "string" does have __concat
```

### Refinement behavior

Unique types can be refined through type guards and pattern matching based on their underlying structural types.

```luau
unique type _itemtag
  type ItemId = _itemtag & string
  type ItemData = _itemtag & { id: string, name: string }

local function processItem(item: ItemId | ItemData)
    if type(item) == "string" then
        -- item is refined to ItemId
        print("Item ID: " .. item)
    else
        -- item is refined to ItemData
        print("Item: " .. item.name)
    end
end
Unique types work with discriminated unions:
luauunique type EventType
type ClickEvent = EventType & { kind: "click", x: number, y: number }
type KeyEvent = EventType & { kind: "key", code: string }

type Event = ClickEvent | KeyEvent

local function handleEvent(event: Event)
    if event.kind == "click" then
        -- event is refined to ClickEvent
        print("Click at", event.x, event.y)
    end
end
```

When multiple unique types are intersected in a discriminated union, refinement preserves only the unique types that are compatible with all variants in the refined branch:

```luau
unique type Validated
unique type Sanitized
unique type EventType

type ValidatedClick = Validated & EventType & { kind: "click", x: number }
type SanitizedKey = Sanitized & EventType & { kind: "key", code: string }
type ValidatedAndSanitizedScroll = Validated & Sanitized & EventType & { kind: "scroll", delta: number }

type Event = ValidatedClick | SanitizedKey | ValidatedAndSanitizedScroll

local function handleEvent(event: Event)
    if event.kind == "click" or event.kind == "scroll" then
        -- event is refined to: Validated & EventType & (ClickEvent | ScrollEvent)
        -- Sanitized is discarded because ValidatedClick doesn't have it
        processValidated(event)  -- Works: both variants have Validated
    end
end
```

# Drawbacks
---
- **Verbose type signatures**: Types that compose unique types will have an ugly type signature (for example `_uniquetype & string`) which means that developers that want a nice clean identifier signature for their unique types (for example `PlayerId`) will not be able to achieve it without risking type safety by directly using unique types instead of composing them, as illustrated here:
```luau
unique type PlayerId -- No type safety, but identifier type signature (signature is just the name which is PlayerId)
unique type _playerid
type PlayerId = _playerid & number -- Has type safety, but ugly type signature
```

- **Naming convention burden**: The recommended pattern of using a private unique type tag (e.g., `_playerid`) composed into a public type alias (e.g., `PlayerId`) creates a burden where developers must choose naming conventions for their tags. This could lead to inconsistency across codebases.

- **Migration complexity**: Existing codebases that have string or number types for IDs will need explicit casts everywhere to convert to unique types, which could be a significant migration effort for large projects.

- **Error message clarity**: Type errors involving unique types may be confusing, especially when the error shows the full intersection type (e.g., `_playerid & number`) rather than the friendly alias (`PlayerId`). This could make debugging harder for developers unfamiliar with unique types.

- **Complexity with multiple unique tags**: Code using multiple intersected unique types (e.g., `Validated & Sanitized & EventType & { ... }`) can become difficult to read and reason about, especially when determining which unique tags are preserved through refinement.

# Alternatives
---

### Just use tables
My example of distinct UserId and AssetId types could instead be written as

```luau
type UserId = { userId: number }
type AssetId = { assetId: number }
```

This works in the current system and makes these types incompatible. For the vectors example, the following snippet would produce the required incompatible types:

```luau
type Vec2 = { x: number, y: number, __isVec2: number }
type Vec3 = { x: number, y: number, z: number, __isVec3: number }
```

A helper type could be used to perform this automatically, such as the following:

```luau
type Tagged<T, Tag> = T & { __tag: Tag }
type UserId = Tagged<number, "UserId">
type AssetId = Tagged<number, "AssetId">
```

In all of these cases, the types are no longer zero-cost at runtime, and in the cases where casting between "nominal" types is desired, it also incurs a runtime cost. The Tagged option does allow runtime introspection, however nothing in this RFC would disallow use of that existing pattern when desired.
