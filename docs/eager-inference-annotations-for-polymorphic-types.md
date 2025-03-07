# Eager Inference Annotations for Polymorphic Types

## Summary  

The RFC introduces a feature to annotate polymorphic function types to express that the first instantiation of a polymorphic type `T` is the one that sticks. 

## Motivation  

The purpose of this feature is to develop syntax to prevent polymorphic types from widening into (e.g., number | string) when a function is implicitly instantiated with different argument types. E.g., `test(1, "a")`. In the following code, Luau's Type Inference Engine V2 infers `T` to be of a union type:

```luau
function test<T>(a: T, b: T): T
    return a
end

local result = test(1, "string") -- inferred type `T`: number | string"
```

This behaviour can be useful in some cases but is undesirable when a polymorphic type is intended to constrain the subsequent input types to be identical to the first usage.

## Design  

We propose adding some symbol as a suffix (or prefix) that annotates the "eager" inference behaviour for a polymorphic type.
Subsequent uses of a polymorphic type `T` where `T` is "eager" will be inferred as the precise type of the first occurrence.

### New Syntax

The `!` syntax modifier would would enforce an eager inference behaviour for `T!`:

```luau
function test<T!>(a: T, b: T): T
    return a
end

test(1, "string") -- TypeError: Expected `number`, got `string`
```

## Drawbacks  

- Introduces a new syntax modifier `!`, which may lead to a symbol soup. However, `T!` doesn't seem too shabby next to `T?`.
- Introduces a simple change to luau's parser, marginally increasing parsing complexity.

## Alternatives  
### Per-usage-bindings
Flip the relationship being declarared per-type-parameter to per-usage which provides more control in expressing the inference, and could allow both instantiation behaviours of polymorphic types under a uniform syntax.

A polymorphic typed marked with type `T!` will not contribute to the instantiation of type `T` in the function. Instead, `T` should be inferred on the arguments without the annotation:

```luau
function test<T>(first: T, second: T, third: T!): T
    return first
end

test(1, "string", true) -- TypeError: Type `boolean` could not be converted into `number | string`
```

This has the added drawback that the `!` syntax modifier would need to be barred from return types, as the return type holds no relevance to implicit instantiation.
Also has the major drawback of symbol complexity. E.g., type aliases with `T!?` are entirely possible under this model.

### noinfer\<T\>
Same as above, except we introduce no new syntax, symbol usage, etc. into the language. Create a binding similar or equivalent to typescript's noinfer<T>:

```luau
function test<T>(first: T, second: T, third: noinfer<T>): T
    return first
end

test(1, "string", true) -- TypeError: Type `boolean` could not be converted into `number | string`
```

### Keywords
Something like `<greedy T>` or `<strict T>` should also be considered if we want to reduce symbols.
