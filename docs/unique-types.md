# Unique Types
---
## Summary
---
This RFC proposes adding support for unique types to luau, which are a way to define nominal types that are subtypes of a supertype, and are able to hold instantiated generic types as metadata.

## Motivation
---
Since Luau uses structural typing, there is no way to make a primitive distinct. If there are two type aliases PlayerId and AssetId and they are both `string`s, the type checker allows a PlayerId to be passed into a function expecting AssetId because they are both just aliases to `string`.

Current workarounds like tagging (string & { _tag: "PlayerId" }) are messy and confuse autocomplete.

Unique types solve this by being completely unique from any other type, therefor allowing programmers to bar casts between different unique types.
A supertype and a list of generics can be assigned to a unique type to alter its subtyping behavior, making it easier to inter-operate between unique types and normal Luau structural types.
A unique type will be able to be cast to its supertype, but not to other unique types or types that are not its supertype.

## Design
---
The proposed syntax to create a unique type is to define it using `type TypeName: Supertype`, the unique type `TypeName` will be defined as having a supertype `Supertype`, defined after the colon. A unique type with no supertype is not allowed as that type would never exist and is "uninhabited".

A unique type is allowed to have other unique types as its supertype

### Casting semantics

When trying to convert into a unique type, a cast will NOT be implicitly performed. An explicit cast must be done first, because unique types are different types from types such as literals and primitives.

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

However, unique type can be implicitly cast out into another type as long as the type is a supertype of it.

```luau
type UniqueType: number

local u = 10 :: UniqueType

local function needsNumber(a: number)

needsNumber(u) -- This is fine! number is a supertype of UniqueType, so an implicit cast is allowed.

local a: number = u -- This is also fine
```

Unique types can be casted to other structural types provided the types are compatible in a structural manner and vice versa. That is to say:

```luau
type Vec2: { x: number, y: number }
type Vec3: { x: number, y: number, z: number }

local vec2_1 = { x=1, y=1 }
local vec3_1 = { x=1, y=1, z=2 }

local vec2_2 = vec3_1 :: Vec2  -- Works, "x" and "y" are present, which is all that's required
local vec3_2 = vec2_1 :: Vec3 -- Doesnt work, "z" is missing from the type
local vec3_3 = vec3_2 :: Vec2 -- Doesnt work, Vec3 cannot be cast into Vec2 despite the fact that Vec2 is a valid subtype of Vec3

local vec2_3 = vec2_2 :: {x: number, y: number} -- This works because {x: number, y: number} is a subtype of {x: number, y: number} (itself)
```

### Operations & interface of a unique type

The operations & interface of a unique type inherit from its defined supertype, as the unique type is gauranteed to have everything that the supertype has.

However, in the case of primitive, aliased or other unique type supertype definitions, all usages of the supertype in the unique type's type signature (including metamethods and operator overloads) should be replaced with the unique type.

The reasoning for this is because primitive, aliased or other unique types can reference themselves in their own definition, so to avoid examples of, for example, adding 2 unique types together and getting a primitive, this replacement of types must be done.

An example with a table literal as a supertype (not primitive, aliased or unique type):
```luau
type Vector4: setmetatable<{x: number, y: number, z: number, w: number}, {
  __add: (Vector4, Vector4) -> (Vector4)
}>

local function Vector4(x: number?, y: number?, z: number?, w: number?): Vector4

local a = Vector4(1, 2, 3, 4)
local b = Vector4(2, 3, 4, 5)

print(a + b) -- Works, since we defined the supertype as a literal it's unecessary to replace anything inside it
```

And in the case of primitive, aliased or other unique type supertype definitions, here for example, the function signature `string.sub(string, number?, number?)` would be replaced with `PlaceId.sub(PlaceId, number?, number?)` in thexe following ample:
```luau
type PlaceId: string

local id1 = "1212" :: PlaceId
local id2 = "32302309" :: PlaceId

local subbed = id1:sub(3, -1) -- This works because the string type in the 1st argument has been replaced with PlaceId
local result = id1..id2 -- The type of result here would be PlaceId because the function signture "__concat: (string, string) -> string" would be replaced with "__concat: (PlaceId, PlaceId) -> PlaceId", this also means you cant concat a PlaceId with any ol' string
```

More examples:
```luau
type RayDirection: vector -- This is a primitive supertype, so any mentions of vector in its type signature should be replaced with RayDirection. The vector type itself does not have any methods, however it does have metamethods as operator overloads so that's where the type signature will be replaced.

local add = (vector.create(10, 2, 2) :: RayDirection) + (vector.create(2, 2, 2) :: RayDirection) -- Works
local dir = vector.create(1, 2, 3) :: RayDirection
local len = vector.magnitude(vector.cross(dir, vector.one)) -- This works because converting out of a unique type is allowed to be done implicitly. However the return type of vector.cross will still be a vector. This MAY be undesirable but highly unlikely so.
```

```luau
type ReadMode: "hi" -- This would be the case of a primitive supertype, why? Because "hi" is a subtype of string, and string is a primitive. So any usage of the string type inside the "hi" literal type would be replaced with ReadMode
```

With aliases:

```luau
type Object = setmetatable<{}, {__index: {new: () -> Object}}>

type MyObject: Object -- This would replace all usages of Object inside the type signature with MyObject for the type signature of MyObject. So in this case that means the function signature of new() in __index is now new: () -> MyObject.
```

It's important to note that library functions would NOT be affected. It is expected for developers to implement their own libraries for manipulating unique types that are subtypes of primitives, for example:

```luau
type ImageBuffer: buffer
type u8: number
type usize: number

local ImageBuffer = {}

function ImageBuffer.writeu8(buf: ImageBuffer, offset: usize, value: u8)
  buffer.writeu8(buf, offset, value)
end

return ImageBuffer
```

There may be more complex examples however this is left as an exercise for the reader.

### Behavior with intersections

Using a unique type in an intersection would simply intersect with the supertype of the unique type, for example:
```luau
type Thing: {a: string}
type ExtendedThing = Thing & {b: number} -- Aliases still work with unique types!
-- The supertype of ExtendedThing has been expanded, and since in the case of intersections, wider = subtype, that means ExtendedThing is now a subtype of {a: string} which is the supertype of Thing.

local thing = {a: string, b: number} :: ExpandedThing -- Works!
local thing2 = thing :: Thing -- This works! Since thing is actually the same unique type, just with the expanded supertype of ExtendedThing, and ExtendedThing is a subtype of {a: string} which is the supertype of Thing, that means this cast is valid.
```

### Behavior with unions

Using a unique type in a union would work, illustrated in something like:

```luau
type UserIdNumber: number
type UserIdString: string

local function getData(id: UserIdNumber | UserIdString) -- This makes sense, UserIdNumber | UserIdString reads as "UserIdNumber, a type that is a subtype of number, or UserIdString, a type that is a subtype of string".
local function getDataStringy(id: string | UserIdString) -- This also makes sense, string | UserIdString reads as "A string, or UserIdString, a type that is a subtype of string".

local data = getData(1234 :: UserIdNumber)
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

However, if a member of a unique type was to be refined like so:

```luau
type Proxy: {inst: Instance?, metadata: {[string]: any}}

local function modify(p: Proxy)
    if p.inst then
        -- The supertype of p would be refined to Proxy & {inst: ~(false?) & Instance, metadata: {[string]: any}}, narrowing the type and becoming a subtype of the original supertype.
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
type Component<T>: i53
type Entity: i53

local function newEntity(): Entity
local function newComponent<T>(): Component<T>
local function get<T>(entity: Entity, component: Component<T>): T

local entity = newEntity()
local component = newComponent<<string>>()

local value = get(entity, component) -- value is string!
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

- Using `type:__eq()` on a unique `type` object on any type other than itself or the type it was instantiated from should return `false` (for example, where T is instantiated from UniqueType and passed as a parameter, `T == UniqueType` -> `true`, `T == T` -> `true`, `UniqueType == types.number` -> `false`, `T == types.string` -> `false`).
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
- Unique types that are used in type functions should have a `.tag` property set to `"unique"` too.
- Implement a new method to `type`, which is `type:generics() -> {type}`. This will return the list of instantiated generic values bound to the unique type.
- Implement a new method to `type`, which is `type:setgenerics({type}) -> ()`. This will set the list of instantiated generic values bound to the unique type. 
- `type:is()` will work if you try to check the supertype of a unique type, so for example:
    ```luau
    type PlayerId<T>: T
    
    type function playerIdToString(t: type)
        assert(t:is("number") and t == PlayerId)
        local newid = PlayerId(types.string)
        return newid
    end

    local a: PlayerId<number>
    local b: playerIdToString<typeof(a)> = tostring(a)
    ```
- `type:issubtypeof(T)` should return true if T is defined as a supertype of the unique type, false if otherwise.

# Drawbacks
---

- Values need to be explicitly cast to the unique type before being able to be assigned to an annotated variable of a unique type
- The introduction of nominal types into the luau type system would increase the complexity of the type system. It adds another thing for beginners to the language to learn, and requires additional work within the type solver.
- Programmers may be confused when mixing structural types and nominal types, especially as they are likely already used to the exclusively-structural nature of luau. Many things that seem to be "correct" from a structural perspective would be disallowed by the type system.

# Alternatives
---

### The nominal typing rfc
There is an alternative nominal typing rfc that proposes encapsulating structural types inside of nominal types instead:
[#123](https://github.com/luau-lang/rfcs/pull/123)

The way this alternative rfc implements nominal types means that you aren't able to do operations such as multiplication on a `type UserId: number` type that results in a `number` type instead of another `UserId` type (desired) by making nominal types completely opaque when it is encapsulating a primitive type, unlike what this rfc proposes (since it relies on subtyping, any operations such as number + number -> number will be passed onto a unique type that uses it as its supertype even though the types are different from it.)

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
