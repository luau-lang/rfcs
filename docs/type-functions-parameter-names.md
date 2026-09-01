# Feature name: Parameter Names in types.newfunction

## Summary
Add the ability to specify parameter names when dynamically constructing function types using the experimental `types.newfunction` API. This will improve code readability, developer experience (DX), and autocomplete integration in tooling such as Luau LSP and Roblox Studio.

## Motivation
Currently, when a developer generates a custom function type using `types.newfunction(...)`, they can only define the types of the parameters, but not their names.

Example of the current limitation:

```luau
local dynamicFuncType = types.newfunction({
    parameters = {
        head = { types.string, types.boolean }
    },
    returns = {
        head = { types.number }
    }
})
```

The resulting type is evaluated as `(string, boolean) -> number`. While functionally correct for type-checking, this significantly degrades the developer experience because users of this type lose context about what the `string` and `boolean` arguments actually represent. 

If we compare this to standard static Luau syntax:

```luau
type MyFunc = (PlayerName: string, IsAdmin: boolean) -> number
```

The parameter names `PlayerName` and `IsAdmin` are preserved and heavily utilized by language servers to provide meaningful autocomplete tooltips and signature help.

Developers increasingly rely on type functions to generate schemas for networking, ECS frameworks, and binary serialization wrappers. Allowing `types.newfunction` to attach names to parameters would bring dynamic type generation on par with static type definitions.

## Design
We propose extending the configuration table accepted by `types.newfunction` to accommodate parameter names alongside parameter types.

Based on the current API where `parameters` is an object containing a `head` array of types, here are the proposed approaches:

### Option 1: Names property in `newfunction`
Add an optional `parameterNames` field to the configuration table.

```luau
return types.newfunction({
    parameters = {
        head = { types.string, types.boolean }
    },
    parameterNames = { "PlayerId", "IsAdmin" },
    returns = {
        head = { types.number }
    }
})
```
The resulting type would be correctly evaluated and displayed by language servers as `(PlayerId: string, IsAdmin: boolean) -> number`.

If the length of the `parameterNames` array does not match the arity of the `parameters.head` table, the type-checker should emit a type error during the type function evaluation to enforce correctness.

### Option 2: Dict-like syntax in `head`
Allow the `head` array to optionally act as a dictionary where the keys act as the parameter names:

```luau
return types.newfunction({
    parameters = {
        head = {
            Var1 = types.boolean,
            Var2 = types.string
        }
    },
    returns = {
        head = { types.number }
    }
})
```
*Note:* Since Luau dictionary order is not guaranteed, this might require compiler-level stability guarantees for keys or could be problematic for strict positional arguments.

### Option 3: Named parameters via Singletons
Use string singletons as keys in the `head` table to represent names:

```luau
return types.newfunction({
    parameters = {
        head = {
            [types.singleton("Var1")] = types.boolean, 
            [types.singleton("Var2")] = types.string
        }
    },
    returns = {
        head = { types.number }
    }
})
```
*Note:* Similar to Option 2, this relies on dictionary key ordering which may not be deterministic.

## Drawbacks
- Increases the complexity of the `types` library API.
- Parameter names in Luau function types do not affect type strictness or execution logic, meaning this change provides no behavioral benefit to the type-checker itself; it is purely a DX and tooling enhancement.
- Options 2 and 3 introduce dictionary keys into `head`, which traditionally expects an ordered array of types. Luau does not guarantee iteration order for non-array keys, which could lead to non-deterministic parameter orders unless specifically handled by the compiler.

## Alternatives
### Do Nothing
Developers will continue to see anonymous `(type1, type2)` signatures in autocomplete for dynamically generated function types, forcing them to rely on external documentation or inline comments to understand the purpose of the arguments.
