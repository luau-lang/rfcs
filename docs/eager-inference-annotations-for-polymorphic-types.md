# Eager Inference Annotations for Polymorphic Types

## Summary  

The RFC introduces a feature to annotate a polymorphic function signature to express that the first instantiation of a polymorphic type T is the one that sticks. 

## Motivation  

The purpose of this feature is to prevent polymorphic types from widening into (e.g., number | string) when the function is called with different arguments, like in the case of `test(1, "a")`. Without the `!` annotation, the solver would infer `T` to be `number | string`, which is undesirable when the intention is to enforce strict type consistency across the function. 

```luau
function test<T>(a: T, b: T): T
    return a
end

local result = test(1, "string") -- inferred type: number | string"
```

This behaviour can be useful in some cases but is undesirable when a function is intended to constrain the type to. An eager binding would prevent `T` from being instantiated with different types and instead produce a type error if `b` does not match `a`.  

## Design  

We propose adding some symbol as a suffix (or prefix) that annotates the inference behaviour for the polymorphic type.

### New Syntax  

The `T!` syntax would would enforce an eager inference behaviour for `T`.

```luau
function test<T!>(a: T, b: T)
end

test(1, "string") -- Type error: Expected `number`, got `string`
```

## Drawbacks  

- Introduces a new syntax modifier (`T!`), which may lead to a symbol soup, but it doesn't seem too shabby next to `T?`.

## Alternatives  
### Function-argument-bindings
Flip the relationship being declarared per-type-parameter to be per-argument which provides more control in expressing the inference and could allow both instantiation  behaviours of polymorphic types under an uniform syntax.

A polymorphic function has arguments marked with T! will not contribute to the inference of `T`. Instead `T` should be inferred on the arguments without the annotation.
```luau
function test<T>(first: T!, second: T!, third: T): T
end

test(1, "string", true) -- Type error: Expected `boolean`, got `number`
```
### Type Function Constraint
Provide a `types.monomorphic<T>` function in user-defined type functions to enforce monomorphism dynamically. But this would probably require some way to propagate an `*error-type*` to the user.
### Keywords
Something like `<greedy T>` or `<strict T>` should also be considered if we want to reduce symbols.


