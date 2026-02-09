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

Unique types solve this by being completely unique from any other type, therefor allowing programmers to bar casts between different unique types.
A supertype and a list of generics can be assigned to a unique type to alter its subtyping behavior, making it easier to inter-operate between unique types and normal Luau structural types.
A unique type will be able to be cast to its supertype, but not to other unique types or types that are not its supertype.

## Design
---
The proposed syntax to create a unique type is to define it using `type TypeName: Supertype`, the unique type `TypeName` will be defined as having a supertype `Supertype`, defined after the colon. A unique type with no supertype is not allowed as that type would never exist and is "uninhabited".

### Behavior with autocomplete

The autocomplete of a unique type should inherit from its defined supertype, as the unique type is gauranteed to have everything that the supertype has.

### Variable definition semantics

When assigning a value to a variable, a cast will NOT be implicitly performed. An explicit cast must be done first, because unique types are different types from types such as literals and primitives.

A unique type cannot be cast to another unique type, however can be cast to types it is subtype of (defined by the type expression after the : in the unique types declaration)
Illustrated in code:

```luau
type UserId: number
type PlaceId: number

local user1: UserId = 2  -- Doesnt work, must cast first
local user2 = 2 :: UserId -- Works! UserId is a subtype of number and its being typecast
local user3 = "2" :: UserId -- Doesnt work, string is not a supertype of UserId
local user4: UserId = 12323 :: PlaceId -- Doesnt work, could not convert PlaceId into UserId
local user5 = 1234 :: PlaceId
local user6 = user5 :: UserId -- Doesnt work, could not convert PlaceId into UserId

local function getPlaceData(id: PlaceId)

local data = getPlaceData(1234) -- Doesnt work, must explicitly cast number to PlaceId
local data = getPlaceData(1234 :: PlaceId) -- Works!

local a = 10
local moredata = getPlaceData(a) -- Again, doesnt work
local moreadaatatata = getPlaceData(a :: PlaceId) -- Works!
```

Some more examples involving more types of literals:

### Behavior with intersections

Using a unique type in an intersection would result in `*error type*`, as a unique type simply denotes a distinct type that is a subtype of something else, and intersecting with that shouldn't be allowed because theres no actual "value" to intersect with here.

### Behavior with unions

Using a unique type in a union would work, illustrated in something like:

```luau
type UserIdNumber: number
type UserIdString: string

local function getData(id: UserIdNumber | UserIdString) end

local data = getData(1234 :: UserIdNumber) -- This makes sense, UserIdNumber | UserIdString reads as "UserIdNumber, a type that is a subtype of number, or UserIdString, a type that is a subtype of string".
```

### Casting semantics
Unique types can be casted to other unique types, or other structural types provided the types are compatible in a structural manner. That is to say:

```luau
type Vec2: { x: number, y: number }
type Vec3: { x: number, y: number, z: number }

local vec2_1 = { x=1, y=1 } :: Vec2
local vec3_1 = { x=1, y=1, z=2 } :: Vec3

local vec2_2: Vec2 = vec3_1  -- Works, "x" and "y" are present, which is all that's required
local vec3_2: Vec3 = vec2_1  -- Doesnt work, "z" is missing from the type
local vec3_3: Vec2 = vec3_2 -- Doesnt work, Vec3 cannot be cast into Vec2 despite the fact that Vec2 is a valid subtype of Vec3
```

### Refinement behavior

Unique types can be refined through type guards and pattern matching based on their supertype.

```luau
type ItemId: string
type ItemData: {data: ItemId, name: string}

local function processItem(item: ItemId | ItemData)
    if type(item) == "string" then
        -- item is refined to ItemId as the only type that is a subtype of string is ItemId, and to satisfy type(item) == "string" the type must be a subtype of string
        print("Item ID: " .. item)
    else
        -- item is refined to ItemData as that's the only other member of the union that's not a subtype of string
        print("Item: " .. item.name)
    end
end
```

Unique types work with discriminated unions, however if a unique type itself is a discriminated union, it will not be able to be decomposed into the correct component as unique types due to the "atomic" nature of nominal types:

```luau
type ClickEvent: {kind: "click", x: number, y: number}
type KeyEvent: {kind: "key", code: string}
type UEvent: ClickEvent | KeyEvent
type Event = ClickEvent | KeyEvent

local function handleEvent(event: Event)
    if event.kind == "click" then
        -- event is refined to ClickEvent
        print("Click at", event.x, event.y)
    end
end

local function handleUEvent(event: UEvent)
  if event.kind == "click" then
    -- UEvent is a subtype of ClickEvent | KeyEvent, which means UEvent is either one of these.
    -- However, UEvent is a unique type, which means it cannot be decomposed any further
    -- Due to this, this means that the "event" variable will have 0 autocomplete (opaque) because it's unclear which one it's supposed to be
    -- So here, no refinement occurs and event remains as UEvent
  end
end
```

### Generic arguments semantics

To accomodate usage of generics, unique types are able to declare a list of generic arguments using the `type TypeName<Arg1, Arg2, Arg3, ...Tuple>: Supertype` syntax, or alternatively `type TypeName<Arg1 = A, Arg2 = B, Arg3 = C, ...Tuple = D...>: Supertype` for generics with default values..

Whenever a unique type is instantiated with a list of generics, these generics become part of the instantiated type and will not be discarded even if the generics aren't used in the supertype, acting sort of like metadata for the instantiated unique type.

It's important to note that an instantiated generic unique type T of unique type A will only be able to be cast to another instatiated generic unique type U of unique type A if the generic values of T are all subtypes of the generic values of U (for instance, `A<string> -> A<"hello">` is invalid as `string` is not a subtype of "hello", however `A<"hello"> -> A<string>` is valid, and so is `A<"hello"> -> A<any>`)

An example of usage:

```luau
type i53: number
type Entity<T>: i53

local entA = 102 :: Entity<string>
local entB = 122 :: Entity<number>

local function get<T>(ent: Entity<T>): T

local value = get(entA) -- string
local value1 = get(entB) -- number
```

Generic arguments can also be used to define the supertype, for example:

```luau
type UserId<ValueType = any>: ValueType

local function getUserId(): UserId
local function saveUserId(id: UserId<string>)

local id: UserId<number> = getUserId()
saveUserId(id) -- Does not work! number is not a subtype of string
saveUserId(id :: UserId<string>) -- Does not work! number is not a subtype of string in the generics list
saveUserId(tostring(id) :: UserId<string>) -- Works! The type signature for tostring is tostring(...any) -> string, and since we now have a string it's able to be converted into UserId<string>.
```

### Type function semantics

Due to the nature of unique types, there would be no way to construct unique types in type functions.

However, since you should be able to input unique types into type functions, or use it as an upvalue, the following are some semantic rules for unique `type` objects:

- Calling `type:__eq()` or using the `==` operator on a unique `type` object on any type other than itself should return `false`.
- There should be a new valid string input to `type:is()`, which is `"unique"`. Illustrated in code:
  ```luau
  type function hi(T)
    if T:is("unique") then
      print("t is unique!")
    else
      print("t is normal :(")
    end
    return T
  end
  ```
- Unique types that are passed into type functions should have a `.tag` property set to `"unique"` too.

# Drawbacks
---

- Values need to be explicitly cast to the unique type before being able to be assigned to an annotated variable of a unique type

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
