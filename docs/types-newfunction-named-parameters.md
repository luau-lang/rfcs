# Parameter Names in `types.newfunction`
## Summary
Add the ability to specify parameter names when dynamically constructing function types using `types.newfunction` and `setparameters`. This will improve code readability and the overall developer experience when working with dynamically generated function types.
## Motivation
Luau already supports named function type arguments in static type annotations, as established by the [Named function type arguments RFC](https://rfcs.luau.org/syntax-named-function-type-args.html).
Being able to add names to parameters defined in user-defined type functions increases the expressiveness of the type system, which is the primary motivation of the original [user-defined type functions RFC](https://github.com/luau-lang/rfcs/pull/45).
### Prior Art

- [Function Parameter Names in User-Defined Type Functions](https://github.com/luau-lang/luau/pull/2001)

Parameter names are stored inside function type descriptions and are surfaced in tooltips and signature help:
```luau
type MyFunc = (PlayerName: string, IsAdmin: boolean) -> number
-- tooltip shows: (PlayerName: string, IsAdmin: boolean) -> number
```
However, when a developer generates a custom function type using `types.newfunction(...)`, they can only define the types of the parameters, but not their names.
Example of the current limitation:
```luau
type function foo(): type
    return types.newfunction(
        { head = { types.string, types.boolean } },
        { head = { types.number } }
    )
end
```
The resulting type is evaluated as `(string, boolean) -> number`. While functionally correct for type-checking, this significantly degrades the developer experience because users of this type lose context about what the `string` and `boolean` arguments actually represent.
Developers increasingly rely on type functions to generate schemas for networking, ECS frameworks, and binary serialization wrappers. Allowing `types.newfunction` to attach names to parameters would bring dynamic type generation on par with static type definitions.
Since the [Named function type arguments RFC](https://rfcs.luau.org/syntax-named-function-type-args.html) is already implemented, the internal infrastructure for storing parameter names in function types already exists. This proposal only requires exposing that capability through the `types` library API.
## Design
We propose extending the parameters and returns tables accepted by `types.newfunction` and `setparameters` / `setreturns` to accommodate parameter names alongside parameter types. The current API signatures are:
```luau
-- Constructor
types.newfunction(parameters, returns, generics)
-- where:
--   parameters: { head: {type}?, tail: type? }
--   returns:    { head: {type}?, tail: type? }
--   generics:   {type}?
-- Instance methods
functiontype:setparameters(head: {type}?, tail: type?)
functiontype:setreturns(head: {type}?, tail: type?)
functiontype:parameters(): { head: {type}?, tail: type? }
functiontype:returns(): { head: {type}?, tail: type? }
```

### I suggest this design:

```luau
types.newfunction(parameters, returns, generics): type
-- where:
--   parameters: { head: {[type]: type}?, tail: type? }
-- Instance methods (updated)
functiontype:setparameters(head: {[type]: type}?, tail: type?)
functiontype:parameters(): { head: {[type]: type}?, tail: type? }
```
When `head` uses singleton keys (`{[type]: type}`), the singleton values act as parameter names and the mapped values act as parameter types. When `head` is a plain array of types (`{type}`), parameters remain anonymous — this is fully backward compatible with the current behavior of `types.newfunction`.
### Examples:
```luau
type function greet(a: type, b: type): type
    assert(a:is("singleton") and b:is("singleton"))
    local params = {
        head = {
            [a] = types.string,
            [b] = types.number,
        },
    }
    local returns = {
        head = { types.string },
    }
    return types.newfunction(params, returns)
end
local fn: greet<"name", "age"> -- (name: string, age: number) -> string
```

```luau
type function foo(): type
    local params = {
        head = {
            [types.singleton("PlayerId")] = types.string,
            [types.singleton("IsAdmin")] = types.boolean,
        },
    }
    local returns = {
        head = { types.number },
    }
    return types.newfunction(params, returns)
end
local fn: foo<> -- (PlayerId: string, IsAdmin: boolean) -> number
```
Anonymous parameters (current behavior, unchanged):
```luau
type function bar(): type
    return types.newfunction(
        { head = { types.string, types.number } },
        { head = { types.boolean } }
    )
end
local fn: bar<> -- (string, number) -> boolean
```
## Drawbacks
- **Making changes to an existing API.**
## Alternatives
### Do Nothing
Developers will continue to see anonymous `(type1, type2)` signatures in autocomplete for dynamically generated function types, forcing them to rely on external documentation or inline comments to understand the purpose of the arguments. As the adoption of user-defined type functions grows, this gap will become increasingly noticeable.
