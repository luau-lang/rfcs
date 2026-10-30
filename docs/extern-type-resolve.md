# Resolve extern types by name in type functions

## Summary

Add `types.resolve(name: string): type` to the type-function library. This function resolves a globally registered extern type by its name and returns the corresponding type value, preserving its nominal identity. This allows user-defined type functions to turn computed names or string singleton type arguments into references to host-provided types.

## Motivation

Type functions can inspect and transform types passed to them, but cannot currently look up an arbitrary extern type using a computed string. `types.singleton("Instance")` produces the string singleton type `"Instance"`, not the extern type `Instance`.

This distinction matters for host environments with large sets of extern types, such as Roblox classes. A type function that receives a class name as a string singleton should be able to return the corresponding class type without maintaining a manually written mapping of every supported name.

The expected outcome is a small lookup API that supports these mappings while retaining the identity and relationships of the original extern types.

This would be especially useful in Roblox, where APIs such as `Instance:IsA(className: string)`, `Instance:FindFirstChildWhichIsA(className: string)` and `Instance:QueryDescendants(query: string)` use strings to identify classes, with specialized type-checking behavior implemented in C++. Resolving these names to extern types would allow user-defined type functions to express similar relationships and derive new types from the properties of named classes.

## Design

The type-function library gains the following function:

```luau
types.resolve(name: string): type
```

`name` is an ordinary string within the type-function runtime. Lookup is case-sensitive and searches the root global type scope for a binding whose underlying type is an extern type. The binding must not require generic type or type-pack arguments.

The returned type represents the existing extern type rather than a newly constructed structural equivalent. Its nominal identity, properties, and inheritance relationships are preserved.

For example, assuming the host registers an extern type named `Instance`:

```luau
type function resolveExtern(name: type): type
    assert(name:is("singleton"), "Expected a string singleton")

    local text = name:value()
    assert(typeof(text) == "string", "Expected a string singleton")

    return types.resolve(text)
end

type ResolvedInstance = resolveExtern<"Instance">
-- ResolvedInstance refers to the same extern type as Instance.
```

Names may also be computed inside a type function before being passed to `types.resolve`.

The function raises a type-function runtime error if the name is unknown, the binding is not an extern type, arguments are invalid, or the type cannot be serialized within the existing type-function limits. It does not return `any`, `unknown`, or `nil` as a fallback. Errors propagate through the existing type-function diagnostic mechanism unless caught with `pcall`.

This proposal deliberately excludes:

- Looking up local aliases, imported aliases, or bindings in child scopes.
- Resolving primitive types such as `"number"`; existing fields such as `types.number` remain available.
- Parsing type expressions such as `"Instance?"`.
- Instantiating generic aliases.
- Creating new extern types or looking up ordinary runtime values.

Local aliases do not shadow this lookup. For example, a local `type Instance = number` does not change the result of `types.resolve("Instance")`.

The API is available only within type functions. It does not add runtime reflection to ordinary Luau scripts or require new syntax.

## Drawbacks

String-based lookup is less discoverable and less refactor-friendly than a direct type reference. Misspellings are diagnosed when the type function is evaluated rather than through ordinary name resolution.

Results depend on the extern types registered by the host. A type function that resolves a Roblox-specific class will not necessarily work in another embedding environment.

Restricting lookup to root-global extern types may be surprising given the general name `resolve`. This restriction needs to be documented clearly. Supporting arbitrary aliases later would require additional decisions about scope, shadowing, generics, and solver dependencies.

Resolving an extern type also incurs serialization work, subject to the same complexity limits as other type-function operations.

## Alternatives

- **Pass the type directly.** A type function can receive `Instance` as a type argument instead of receiving `"Instance"`. This is preferable when the type is known statically, but does not support names computed within a type function.
- **Maintain an explicit mapping.** Users can write a table or conditional mapping names to available type values. This avoids a new API but requires maintenance as the host's extern types change.
- **Use `types.find(name): type?`.** Returning `nil` would support optional lookup without `pcall`, but requires callers to handle failure explicitly before returning a type. This proposal instead follows an error-producing `resolve` contract.
- **Use a narrower name such as `types.resolveextern`.** This makes the initial scope clearer, at the cost of a longer name and less room for future expansion.
- **Resolve arbitrary visible aliases.** This is more general but requires defining whether lookup uses the type function's definition scope or its invocation scope, and how unresolved or generic aliases participate in constraint solving.
- **Parse complete type expressions.** Supporting strings such as `"Instance?"` would require parsing and resolving annotations, introducing substantially more complexity than name lookup.

Without this feature, users must pass types directly or maintain their own name-to-type mappings.

## Prior Art

Luau's existing `types` library provides primitive type values, constructors such as `types.singleton`, and operations for inspecting types. `types.resolve` complements these APIs by retrieving an existing named extern type rather than constructing a new type.

TypeScript can express a related mapping using an explicit interface and indexed access types, for example `Registry[Name]`. That approach requires a declared registry; this proposal instead uses the extern bindings already supplied by the Luau host.

Runtime reflection APIs such as .NET's `Type.GetType` also retrieve type representations using strings.
