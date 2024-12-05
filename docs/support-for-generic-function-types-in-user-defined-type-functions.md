# Support for Generic Types and Packs in User-Defined Type Functions

## Summary

Add support for generic function types in user-defined type function to allow more types to be accepted by them, including tables and classes which might contain generic function types.

## Motivation

Generic functions are very common in Luau, especially when inferred by the typechecking engine. They are also often present in classes.

Omitting generic types from appearing in user-defined type functions limits their use.
Even if there is no direct manipulation of generics, an serialization error appears immediately.

## Design

We will introduce a single 'generic' kind for both generic types and generic packs in runtime.

While this doesn't match up the typechecking engine representation directly, we avoid having to deal with two different userdata types. Being opaque about the contents, assigning separate tags is also not that different.

We already have variadic types that get represented as a regular runtime type in the 'tail' table member/arguments.
We also have a different representation of a table with a metatable.

Generics are created by specifying a name and whether or not it is a pack.
Generics that are coming from type inference might not have a display name, but still hold an auto-generated name internally.

Names are the way generic identity is specified.
Generic types are immutable and two generics with the same name are equal, even if they are defined in different function type scopes.

For example, consider a type `<T, U>(T, U, <T>(T) -> U) -> ()`

From the typechecking perspective, inner and outer function 'T' types are different, but they are still represented by the same runtime generic named 'T'.

The ambiguity is resolved purely on the scoping of the generic in question.
When runtime type is returned back to the typechecking engine, only those generics that are in current set of function type generic lists are allowed to be mentioned.
When inner generic function list places name 'T' again, it represents a different typechecking type and hides outer 'T'.

So we can say generics are defined by the lexical scope and that actually matches how they are written out in code.

Just like in the syntax of the language, reusing the same generic name, even between types and packs in a single list in a duplicate generic definition error.

### Examples

Creating a function type with generic types and packs:

```lua
type function pass()
    local T = types.generic("T")
    local Us, Vs = types.generic("U", true), types.generic("V", true)

    local f = types.newfunction()
    f:setparameters({T}, Us);
    f:setreturns({T}, Vs);
    f:setgenerics({T, Us, Vs});
    return f
end

type X = pass<> -- <T, U..., V...>(T, U...) -> (T, V...)
```

Creating a function type with a nested function type that reuses a generic name:

```lua
type function pass()
    local T, U = types.generic("T"), types.generic("U")

    -- <T>(T) -> ()
    local func = types.newfunction({ head = {T} }, {}, {T});

    -- { x: <T>(T) -> (), y: U }
    local tbl = types.newtable({ [types.singleton("x")] = func, [types.singleton("y")] = U })

    -- <T, U>(T, { x: <T>(T) -> (), y: U }) -> ()
    return types.newfunction({ head = {T, tbl} }, {}, {T, U})
end
```

## Updates to the library and type userdata

### `types` Library

| New/Update | Library Functions | Return Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New |  `generic(name: string, ispack: boolean?)` | `type` | returns an immutable instance of a generic type or a pack; name cannot be empty |
| Update | `newfunction(parameters: { head: {type}?, tail: type? }, returns: { head: {type}?, tail: type? }, generics: {type}?)` | `type` | New `generic` argument is added to specify generic types and packs in a single list |

### `type` Instance

| New/Update | Instance Properties | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| Update | `tag` | `"nil" \| "unknown" \| "never" \| "any" \| "boolean" \| "number" \| "string" \| "singleton" \| "negation" \| "union" \| "intersection" \| "table" \| "function" \| "class" \| "thread" \| "buffer" \| "generic"` | Added `generic` as a possible tag |

#### Generic `type` instance

| New/Update | Instance Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New | `name()` | `string?` | name of the generic; `nil` for inferred generics |
| New | `ispack()` | `boolean` | `true` if the generic represents a pack, `false` otherwise |

#### Function `type` instance

| New/Update | Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New | `setgenerics({type}?)` | `()` | sets the function's generic types and packs list; types cannot follow a pack |
| New | `generics()` | `{type}` | returns the function's generic types and packs (in that order in the array) |

## Alternatives

We can assign a different `tag` to a generic pack instead of the `ispack` property.

It is also possible to split generic type and packs list into two.
This was done in an initial prototype implementation as was inconvenient to specify.
One benefit was that you didn't have to split them manually when reading them.

## Drawbacks

Generics do bring additional complexity to user-defined type functions, especially with the way we can now hold a generic type pack as a separate item (while `type T = U...` impossible to define in the syntax of the language).

The way they bring in lexical scoping consideration is also new for user-defined type functions.
