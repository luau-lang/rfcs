# Nominal Typing
## Summary
This RFC proposes adding support for nominal typing to luau, alongside the existing structural typing.

Nominal types are an extension to the type checker, and make no changes to runtime semantics.

## Motivation
Luau's type system is currently entirely structural. This means types that share the same structure, but differ in name, are still considered compatible.

This further extends to types that are not directly equal, but compatible, such as how `type Vec3 = { x: number, y: number, z: number }` can be used in places where a `type Vec2 = { x: number, y: number }` is expected.

Nominal types allow the programmer to state that, for example, these `Vec2` and `Vec3` types are not compatible with each other. Further, two nominal types can share identical structures. This allows for coincidentally overlapping structures to be considered distinct types, such as an overlap between a `Position2D` and `Velocity2D` type both being `{ x: number, y: number }`.

Additionally, nominal types allow for the type system to describe primitive types that have been verified in some way. We could consider the following as an example of this:

```lua
distinct type PositiveNumber = number

function assertPositive(x: number): PositiveNumber
    assert(x >= 0, "not a positive number")
    return x :: PositiveNumber
end
```

Another example use-case is for when working with references to data. We can consider a `distinct type UserId = number` and a `distinct type AssetId = number`, which now disallows the use of asset IDs in places where a user ID was expected.

As a final motivating use-case, classes (and just generally complex tables with metatables) in luau currently expose their full type to users when viewing types in their code editor, notably also in a primitive form with all original type names expanded to their value. This produces very unreadable types that generally end up truncated. Nominal types would instead only show the names of the types (this could potentially be exposed as an option for users of Luau-Analyze, such that the expanded types could be shown when holding a key such as Shift) producing far more readable tooltip information for developers.

## Design
The proposed syntax to mark a type as nominal is to define it using `['export'] ['distinct'] 'type' NAME ...` and `['export'] ['distinct'] 'type' function NAME ...` for standard types and function types respectively. This can be seen in use during the examples given during the Motivation section.

The name `distinct` was chosen as it clearly portrays the behaviour of a nominal type without requiring programmers to know what a `nominal` is. The Alternative section discusses some other potential names.

Nominal types, despite the name, are not matched based on their name. If two source files define a nominal type with the same name, they are incompatible. Sharing a nominal type across files requires exporting it and requiring it in other places. There is no proposed method to avoid this and define two identical types in differing files instead.

### Behaviour with literals
When assigning a literal value to a variable, a cast will not be implicitly performed. It is expected that nominal types will almost exclusively be generated within APIs, rather than written as literals. Illustrated in code:

```lua
distinct type UserId = number
local user1: UserId = 1  -- Not valid
local user2: UserId = 2 :: UserId  -- Valid
```

### Casting semantics
Nominal types can be casted to other nominal types, or other structural types provided the types are compatible in a structural manner. That is to say:

```lua
distinct type UserId = number
distinct type PlaceId = number
type AssetId = number

local asset_1: AssetId = 1
local place_2: PlaceId = 2 :: PlaceId

local user_1: UserId = asset_1  -- Not valid
local user_1: UserId = asset_1 :: UserId  -- Valid

local user_2: UserId = place_2  -- Not valid, nominal type conflict
local user_2: UserId = place_2 :: UserId  -- Valid, as the nominal types are compatible
```

As a further example:

```lua
distinct type Vec2 = { x: number, y: number }
distinct type Vec3 = { x: number, y: number, z: number }

local vec2_1 = { x=1, y=1 } :: Vec2
local vec3_1 = { x=1, y=1, z=2 } :: Vec3

local vec2_2: Vec2 = vec3_1 :: Vec2  -- Valid. "x" and "y" are present, which is all that's required
local vec3_2: Vec3 = vec2_1 :: Vec3  -- Not valid. "z" is missing from the type
```

### Operations on nominal types
Nominal types of `number` and `string` do not allow manipulation using common operators such as `+` and `..`. The following code would be considered a type error:

```lua
distinct type UserId = number
local user_1: UserId = 1 :: UserId
local user_2 = user_1 + 1  -- Cannot perform arithmetic on nominal type
```

In cases where this behaviour would be desired, the option remains to write `(user_1 :: number) + 1` following the previous rules regarding casting. `user_1 + user_1` would likewise be disallowed. This is done to avoid accidental loss of semantic meaning

### Intersection of nominal types
Intersection between two different nominal types always results in a `never` type.

Intersection between any nominal type and itself results in itself.

Intersection between a nominal type and a structural type results in one of:

1. `never`, if the structural constraints cannot be entirely satisfied by the nominal type
2. The nominal type, if the structural constraints completely match the nominal type
3. A `T & { ... }` type, narrowing the nominal type while retaining the nominal semantics.

(3) can only be simplified into (2), in the case where the structural type performs no narrowing and matches the nominal entirely. (3) can never simplify into a purely structural type without the use of an explicit cast.

Nominal types that contain type parameters can have their type parameters narrowed, provided the original nominal type is still present in the resultant type.

To present this as a series of examples:

```lua
distinct type Ok<T> = { ok: true, value: T}
distinct type Err<E> = { ok: false, error: E}
distinct type ResultN<T, E> = Ok<T> | Err<E>
type Result<T, E> = Ok<T> | Err<E>

distinct type OkNS = Ok<number | string>
type ResultNS = OkNS | Err<string>

function foo(x: ResultN<number | string, string>)
    if x.ok then
        -- x : ResultN<number | string, string> & { ok: true }

        -- x does not narrow to Ok<T>, as ResultN is nominal and cannot be discarded.
        -- It however also doesn't narrow to never, as it is possible to satisfy { ok: true } as a constrained version of the nominal
    end
end

function bar(x: ResultN<number | string, string>)
    if x.ok == "bar" then
        -- x : ResultN<number | string, string> & { ok: "bar" }
        -- x : never

        -- x narrows to never, as there is no way for the nominal type to ever satisfy { ok: "bar" }
    end
end

function buzz(x: Result<number | string, string>)
    if x.ok then
        -- x : Result<number | string, string> & { ok: true }
        -- x : Ok<number | string>

        -- Our Result type used isn't nominal, so standard narrowing occurs here

        if typeof(x.value) == "number" then
            -- x : Ok<number>

            -- The generic parameter within Ok has been narrowed, while retaining the nominal type
        end
    end
end

function baz(x: ResultNS)
    if x.ok then
        -- x : OkNS

        -- As above, our Result type used isn't nominal, so standard narrowing occurs here

        if typeof(x.value) == "number" then
            -- x: OkNS & { value: number }

            -- As our OkNS nominal type has encapsulated the union within itself, we cannot narrow the type parameters, nor simplify this intersection type.
            -- x.value specifically is however narrowed to a number by the intersection
        end
    end
end
```

### Nominal function types
Consider the following nominal function type:
```lua
distinct type function Callback = (number) -> boolean

function foo(cb: Callback)
    -- [snip]
end
```

This nominal type disallows the use of `foo(function(_) return true end)`, however `foo((function(_) return true end) :: Callback)` would be allowed. This is identical semantics to non-function nominals, however mentioned explicitly for clarity.

### Runtime semantics
As with all other types, nominal types are erased at runtime. There is no change to the runtime implementation nor any way to access the name of a type from within code.

The `typeof` function will return the runtime type (`number`, `string`, `table`, etc.) rather than the nominal type's name.

## Drawbacks
The introduction of nominal types into the luau type system would increase the complexity of the type system. It adds another thing for beginners to the language to learn, and requires additional work within the type solver.

Making distinction between nominal types in different source files with a shared name may increase complexity in the way the type solver is able to resolve types.

Library developers may over-use nominal types, leading to increased frustration for consumers as casts and constructors are now required in many places they would not have previously been.

Programmers may be confused when mixing structural types and nominal types, especially as they are likely already used to the exclusively-structural nature of luau. Many things that seem to be "correct" from a structural perspective would be disallowed by the type system.

## Alternatives
### Syntax alternatives
A number of different options for the syntax were considered, including (but not limited to):

```lua
type named Foo = number
```
Rejected as `named` suggests only the name is used for type matching.

---

```lua
type unique Foo = number
```
Rejected as a goal was to produce something human readable. `export type unique Foo` is starting to read like a Java function declaration and that's not ideal.

---

```lua
unique type Foo = number
```
The most viable candidate here, though "distinct" was chosen for clarity. Two nominal types can both be a `number` and that doesn't feel "unique".

---

```lua
newtype Foo = number
```
Introduces a variation on the existing "type" syntax, rather than extending it. The chosen syntax also leaves the door over to chain any other future modifiers.

Describing nominal types as a "new type" may be confused as writing `type Foo = number` already reads as "making a new type".

---

```lua
type Foo = number as Foo
```
Requires duplication of the type name, and raises questions such as what `{ x: number as Foo }` or `type Foo = number as Bar` would evaluate as.

---

```lua
type Foo = distinct<number>
```

Making a type nominal is not a type function, and likewise raises questions regarding `{ x: distinct<number> }`

---

```lua
type Foo = newtype Foo
```

Again raises questions regarding `{ x: newtype number }`. Keywords after the `=` sign is confusing and not convention anywhere else.

---

```lua
@distinct
type Foo = number
```

Attributes would be a potentially viable option, though they currently can apply only to functions. This would deviate from the current pattern of `type` being modified by keywords (`export type`). The use of attributes to fundamentally alter the construction of types may also be confusing, as current uses of attributes are as ways to hint behaviour to luau ("make this function native" and "this is a deprecated feature").

### Just use tables
Our example of distinct `UserId` and `AssetId` types could instead be written as

```lua
type UserId = { userId: number }
type AssetId = { assetId: number }
```

This works in the current system and makes these types incompatible. For the vectors example, the following snippet would produce the required incompatible types:

```lua
type Vec2 = { x: number, y: number, __isVec2: number }
type Vec3 = { x: number, y: number, z: number, __isVec3: number }
```

A helper type could be used to perform this automatically, such as the following:

```lua
type Tagged<T, Tag> = T & { __tag: Tag }
type UserId = Tagged<number, "UserId">
type AssetId = Tagged<number, "AssetId">
```

In all of these cases, the types are no longer zero-cost at runtime, and in the cases where casting between "nominal" types is desired, it also incurs a runtime cost. The `Tagged` option does allow runtime introspection, however nothing in this RFC would disallow use of that existing pattern when desired.
