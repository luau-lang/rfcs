# Eager Inference Annotations for Polymorphic Types

## Summary  

Introduce a way to annotate a polymorphic function signature to express that it will only allow one argument type to interact with automatic instantiation.

## Motivation  

Currently, Luau's type solver allows polymorphic types to widen into unions when they are instantiated with multiple types. For example:  

```luau
function test<T>(a: T, b: T): T
    return a
end

local result = test(1, "string") -- inferred type: 1 | "string"
```

This behaviour can be useful in some cases but is undesirable when a function is intended to enforce strict monomorphism. A monomorphic binding would prevent `T` from being instantiated with different types and instead produce a type error if `b` does not match `a`.  

```luau
function test<T!>(a: T, b: T)
end

test(1, "string") -- Type error: Expected `number`, got `string`
```

## Design  

This RFC proposes adding some symbol as a suffix (or prefix) that annotates the inference behaviour for the polymorphic type.

### New Syntax  

The `T!` syntax would indicate a that the type binding for `T` is monomorphic.

### Type Checking Rules  

1. When a polymorphic type is marked as `T!`:
   - The first instantiation of `T` determines its type.
   - Any subsequent use of `T` in the same context must match this type exactly.
   - If a different type is encountered, a **type error** is raised.
   - `T` will not expand into a union.

2. **Behavior in Unions**  
   - A function or type with `T!` cannot instantiate `T` with a union.
   - If `T` is already a union, it must remain a union as new types cannot be added to it.

## Drawbacks  

- Introduces a new syntax modifier (`T!`), which may lead to a symbol soup but it doesn't seem too shabby next to `T?`.

## Alternatives  

- **Type Function Constraint**: Provide a `types.monomorphic<T>` function in user-defined type functions to enforce monomorphism dynamically. But this would probably require some way to propagate an `*error-type*` to the user.
- **Keywords**: Something like `<greedy T>` or `<strict T>` should also be considered if we want to reduce symbols.
- **Function-argument-bindings**: Flip the relationship from being declared per-type-parameter to be set per-argument:<br>`function test<T>(a: T, b: T, c: T!)` which provides the user more control, and could allow both instantiation behaviours of polymorphic types under a uniform syntax.
