# Distinct types (newtypes)

## Summary

Add a built-in type function called `newtype<T>` that will create a type that is nominally distinct from its underlying representation.

## Motivation

Since Luau uses structural typing, there is no way to make a primitive distinct. If there are two types `PlayerId` and `AssetId` and they are both strings, the type checker allows a `PlayerId` to be passed into a function expecting `AssetId` because they are both just `string`.

Current workarounds like tagging (`string & { _tag: "PlayerId" }`) are messy and confuse autocomplete.

## Design

### Syntax

Introduce a `newtype<T>` type function.

```luau
-- These are only compatible with instances of themselves
type PlayerId = newtype<string>
type PlaceId = newtype<string>
```

This also has a matching types library method (`types.newtype(type)`) which can be utilized within type functions.

### Usage

```luau
local id = "player_1234" :: PlayerId
local badId = 123 :: PlayerId -- Cannot convert a number into a PlayerId (newtype<string>)
```

The new type is opaque, it doesn't act like the underlying primitive unless explicitly cast back.

```luau
local function teleport(player: PlayerId, place: PlaceId) end

local player: PlayerId
local place: PlaceId

teleport(place, player) -- Error: place is not a PlayerId
```

```luau
local id = "player_1234" :: PlayerId

local upper = (id :: string):upper() -- Must cast to use string method
local bad = id:upper() -- Error: PlayerId does not have upper
```

## Drawbacks

You must cast to create the value and cast back to use it as its original type.

When you import a distinct type, it is possible to cast it to any value, creating a 'fake' value that does not strictly adhere to the intended domain constraints of that type.

Modules exporting `newtype` definitions will likely need to provide "constructor" functions to ensure values are properly validated.

## Alternatives

### Value level constructor

We could allow the type name to be used as a function call to "wrap" the primitive such as `local id = PlayerId("user_1234")` and remove support for casting completely.

This is not feasible as the type namespace and value namespace is strictly separted; implementing this would require significant changes to the compiler.

### Subtyping

`newtype<T>` could be a subtype of `T`, which would allow a `PlayerId` to be passed into a function expecting a `string` without a cast, while preventing a raw `string` from being passed into a function expecting a `PlayerId`.

This is undesirable as using operators on two different distinct types with the same underlying type would produce a result but would be logically incorrect (`Distance + Time`) which invalidates the point of distinct types.

### Argument-less `newtype` wtih intersection

An alternative design is to make `newtype` a type function that takes no arguments, returning a unique, empty nominal type. To associate it with a representation like `string`, the user would use an intersection.

```luau
type PlayerId = newtype<> & string
type PlaceId = newtype<> & string
```

This mirrors the current "tagging" workaround but uses a native language feature instead of phantom table fields.
