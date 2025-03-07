# Eager Inference Annotations for Polymorphic Types

## Summary  

The RFC introduces a feature to annotate polymorphic function types to express that the first instantiation of a polymorphic type `T` is the one that sticks. 

## Motivation  

The purpose of this feature is to dvelop syntax to prevent polymorphic types from widening into (e.g., number | string) when a function is implicitly instantiated with different argument types. E.g., `test(1, "a")`. In the following code, Luau's current solver infers `T` to be of a union type:

```luau
function test<T>(a: T, b: T): T
    return a
end

local result = test(1, "string") -- inferred type `T`: number | string"
```

This behaviour can be useful in some cases but is undesirable when a polymorphic function is intended to constrain the input types to be consistent.

## Design  

We propose adding some symbol as a suffix (or prefix) that annotates the "eager" inference behaviour for a polymorphic type.
Subsequent usages of type `T` where `T` is "eager" would be ignored during instantiation.

### New Syntax

The `!` syntax modifier would would enforce an eager inference behaviour for `T!`:

```luau
function test<T!>(a: T, b: T): T
    return a
end

test(1, "string") -- TypeError: Expected `number`, got `string`
```

## Drawbacks  

- Introduces a new syntax modifier (`T!`), which may lead to a symbol soup, but it doesn't seem too shabby next to `T?`.
- Introduces a simple change to luau's parser, marginally increasing parsing complexity.

## Alternatives  
### Function-argument-bindings
Flip the relationship being declarared per-type-parameter to per-function-argument which provides more control in expressing the inference, and could allow both instantiation behaviours of polymorphic types under a uniform syntax.

A polymorphic function's arguments marked with type `T!` will not contribute to the instantiation of type `T` in the function. Instead, `T` should be inferred on the arguments without the annotation:

```luau
function test<T>(first: T, second: T, third: T!): T
end

test(1, "string", true) -- TypeError: Type `boolean` could not be converted into `number | string`
```

This has the added drawback that the `!` syntax modifier would need to be barred from return types, as the return type holds no relevance to implicit instantiation.


### Keywords
Something like `<greedy T>` or `<strict T>` should also be considered if we want to reduce symbols. This idea has merit when considering the potential complexity of type aliases combined with `T!?`
