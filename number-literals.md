# Numeric Type Literals as Concrete Table Property Keys

## Dependency

1. [Singleton Type Literals as Table Property Keys](singleton-indexer-to-property)

## Summary

This proposal introduces support for numeric type literals as concrete table property keys, analogous to the existing treatment of string literals.

The goal is to allow numeric singleton types to represent specific, statically known table properties in type definitions. This would extend the existing literal-property model from string keys to numeric keys while preserving the existing semantics of numeric indexers.

In particular, a numeric literal type such as:

```luau
type N = 1337
```

should be usable as a concrete table property key:

```luau
type B = {
    [N]: number,
}
```

This should be semantically equivalent to:

```luau
type B = {
    [1337]: number,
}
```

The proposal is intended to provide a type-level representation for statically known numeric indices, which is particularly useful for structures based on arrays of arrays (AoA), heterogeneous tuples, and other data structures where the type of a value depends on its exact numeric index.

## Motivation

Luau already provides a natural type-level representation for tables whose properties are identified by string literals:

```luau
type A = {
    foo: () -> (),
    bar: () -> never,
    too: number,
    hi: A,
}
```

This describes a table whose keys are known string literals and whose values may have different types.

The language also provides generic array-like table types:

```luau
type B<T = unknown> = {T}
```

which is equivalent to:

```luau
type B<T = unknown> = {[number]: T}
```

This representation is appropriate when all numeric indices share a common value type.

However, some implementations require the type of a value to depend on its exact numeric index.

For example, consider a runtime structure such as:

```luau
local A = {
    {
        [1] = 0,
        [2] = "",
        [3] = {
            hi = function()
                return "hi"
            end,
        },
    },
}
```

The table contains different value types at different numeric indices:

```text
index 1 -> number
index 2 -> string
index 3 -> { hi: () -> string }
```

Representing this structure using only:

```luau
[number]: T
```

is insufficient because a numeric indexer describes a common type for an arbitrary set of numeric keys. It cannot express that index `1` has one type while index `2` and index `3` have different types.

The desired type-level representation is instead something equivalent to:

```luau
type A = {
    [1]: number,
    [2]: string,
    [3]: {
        hi: () -> string,
    },
}
```

This distinction is particularly important for implementations of heterogeneous array-like structures and AoA-style data representations, where a specific index has semantic meaning.

Numeric singleton types provide a natural mechanism for expressing these properties:

```luau
type First = 1
type Second = 2
type Third = 3
```

The proposal therefore extends the existing literal-property model so that numeric singleton types can represent concrete numeric properties in the same way that string singleton types represent concrete string properties.

### Relation to Type Packs

This proposal is also motivated by the current limitations around type packs.

Type packs provide a mechanism for representing a sequence of types, but they cannot currently be used directly as a description of a heterogeneous numeric table such as:

```luau
{T...}
```

where each element of the resulting table would correspond to a specific numeric index and retain its individual type.

For example, conceptually:

```luau
type C<T, U...> = {
    [1]: number,
    [2]: T,
    [3]: {U...},
}
```

should allow the numeric structure of the table to preserve the correspondence between indices and types.

The exact interaction between numeric literal properties and type-pack expansion may require additional specification, but numeric concrete properties provide the necessary foundation for such representations.

## Design

### Syntax

Numeric literals should be allowed in table type property declarations in the same manner as existing literal keys.

For example:

```luau
type C<T, U...> = {
    [1]: number,
    [2]: T,
    [3]: {U...},
}
```

Here:

```text
1 -> number
2 -> T
3 -> {U...}
```

The numeric keys are concrete properties rather than a general numeric indexer.

This corresponds to the existing runtime syntax:

```luau
local A = {
    {
        [1] = 0,
        [2] = "",
        [3] = {
            hi = function()
                return "hi"
            end,
        },
    },
}
```

The type system would therefore be able to describe the concrete structure of the runtime table rather than reducing all numeric keys to a single `[number]` indexer.

### Numeric Literal Types

A numeric literal can also be represented through a type alias:

```luau
type N = 1337

type B = {
    [N]: number,
}
```

This should be equivalent to:

```luau
type B = {
    [1337]: number,
}
```

The alias does not introduce a new runtime concept. `N` simply resolves to the numeric singleton type `1337`, which represents exactly one possible table key.

The semantic distinction is therefore:

```text
numeric singleton
    -> concrete numeric property

number
    -> numeric indexer
```

For example:

```luau
type Index = 1

type T = {
    [Index]: string,
}
```

represents:

```text
property:
    1 -> string
```

whereas:

```luau
type Index = number

type T = {
    [Index]: string,
}
```

represents:

```text
indexer:
    number -> string
```

### Concrete Numeric Properties vs. Numeric Indexers

The proposal maintains the distinction between a known numeric key and an arbitrary numeric key.

For example:

```luau
type T = {
    [1]: string,
    [2]: number,
    [3]: boolean,
}
```

represents three concrete properties:

```text
1 -> string
2 -> number
3 -> boolean
```

It is fundamentally different from:

```luau
type T = {
    [number]: string,
}
```

which represents:

```text
number -> string
```

The latter places a common constraint on an arbitrary numeric key, while the former describes the exact shape of a heterogeneous numeric table.

### Type Alias Resolution

Numeric singleton aliases should resolve in the same manner as other singleton aliases.

For example:

```luau
type First = 1
type Second = First

type T = {
    [Second]: number,
}
```

should have the same semantics as:

```luau
type T = {
    [1]: number,
}
```

The semantic classification therefore happens after resolving the type used as the key.

### Interaction with Existing Array Types

The proposal does not replace or alter the existing meaning of array types.

For example:

```luau
type T = {string}
```

continues to represent an array-like table whose numeric values have a common type.

Likewise:

```luau
type T = {[number]: string}
```

continues to represent a numeric indexer.

Concrete numeric properties are an additional capability for expressing heterogeneous structures:

```luau
type T = {
    [1]: number,
    [2]: string,
}
```

The two models can coexist.

For example:

```luau
type T = {
    [1]: number,
    [2]: string,
    [number]: unknown,
}
```

contains both concrete properties and a general numeric indexer.

When a concrete numeric property is accessed, the concrete property's type should take precedence over the general numeric indexer according to the existing model for concrete properties and indexers.

Thus:

```text
T[1] -> number
T[2] -> string
T[otherNumber] -> unknown
```

The proposal does not otherwise modify the interaction between properties and indexers.

## Type Packs

Numeric concrete properties also provide a natural basis for representing heterogeneous positional structures.

Conceptually, a type pack:

```luau
type T...
```

represents an ordered sequence of types. A corresponding heterogeneous table could associate each element of that sequence with a concrete numeric index.

For example, given a conceptual pack:

```text
T... = number, string, boolean
```

the corresponding table shape could be represented as:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

This is different from:

```luau
{number | string | boolean}
```

because the latter does not preserve the relationship between a specific index and its type.

The proposal therefore provides an important type-system primitive for future or accompanying work on expressing type-pack-backed heterogeneous tables.

The exact syntax and semantics for expanding a type pack into numeric properties are outside the core of this proposal and should be specified separately.

## Numeric Singleton Resolution

A numeric type should be treated as a concrete property key only when it represents exactly one numeric value.

For example:

```luau
type A = 1
type B = 2
type C = 1337
```

are all concrete numeric keys.

In contrast:

```luau
type A = number
```

represents an arbitrary numeric key and therefore remains an indexer.

The distinction can be summarized as:

```text
singleton numeric type
    -> one known property

non-singleton numeric type
    -> numeric indexer
```

This is consistent with the distinction between concrete properties and indexers elsewhere in the type system.

## Duplicate Numeric Keys

Numeric properties should follow the existing rules for duplicate concrete properties.

For example:

```luau
type T = {
    [1]: number,
    [1]: string,
}
```

contains two declarations referring to the same concrete property key.

Likewise:

```luau
type N = 1

type T = {
    [1]: number,
    [N]: string,
}
```

both declarations refer to the same property.

The proposal does not introduce a new duplicate-property model. Existing rules for repeated property declarations should determine whether such declarations are rejected or combined.

## Unions

The proposal does not require unions of numeric literals to be expanded into multiple concrete properties.

For example:

```luau
type Keys = 1 | 2

type T = {
    [Keys]: string,
}
```

should not automatically be interpreted as:

```luau
type T = {
    [1]: string,
    [2]: string,
}
```

A union represents multiple possible values rather than one concrete key.

Therefore, the core semantic rule remains:

```text
one singleton value
    -> one concrete property

multiple possible values
    -> not a concrete singleton property
```

Any future semantics for expanding literal unions into multiple concrete properties should be considered separately.

## Runtime Considerations

The proposal is entirely a type-system feature.

No changes to the Lua VM or runtime table representation are required.

For example:

```luau
type N = 1337

type T = {
    [N]: number,
}
```

still corresponds to an ordinary runtime table:

```luau
local value = {
    [1337] = 10,
}
```

The type alias and numeric singleton type have no separate runtime representation.

The type checker only gains the ability to recognize that `N` denotes one concrete numeric key.

## Backwards Compatibility

Existing numeric indexers retain their current semantics.

For example:

```luau
type T = {
    [number]: string,
}
```

continues to represent a numeric indexer.

Similarly:

```luau
type T = {
    [N]: string,
}
```

where:

```luau
type N = number
```

continues to represent:

```luau
type T = {
    [number]: string,
}
```

The semantic change only applies when the resolved key type is a numeric singleton.

Code that currently relies on a numeric singleton being interpreted as an indexer would therefore change meaning. However, such a type describes exactly one possible key and is consequently more naturally modeled as a concrete property.

## Drawbacks

The primary drawback is that the meaning of:

```luau
[T]: U
```

depends on the resolved type of `T`.

For example:

```luau
type A = 1
type B = number
```

produce different table type constructs:

```luau
type T = {
    [A]: string,
}
```

represents a concrete property, while:

```luau
type T = {
    [B]: string,
}
```

represents an indexer.

This introduces semantic complexity into the type checker because the classification of the table entry cannot be determined solely from its syntax.

However, this distinction is already inherent in the type system's distinction between singleton and non-singleton types and is necessary to represent concrete numeric properties through aliases.

A second potential drawback is increased complexity around unions, type packs, and other composite types. The core proposal intentionally does not define automatic expansion of such types into multiple properties. These cases can be specified independently without complicating the fundamental singleton-property rule.

## Alternatives

### Do Not Support Numeric Concrete Properties

The simplest alternative is to retain only the existing numeric indexer:

```luau
type T = {
    [number]: unknown,
}
```

This is sufficient for homogeneous arrays but cannot precisely describe heterogeneous numeric tables.

For example, it cannot express that:

```text
T[1] -> number
T[2] -> string
T[3] -> SomeOtherType
```

without losing the relationship between a specific numeric index and its corresponding type.

This limitation becomes significant for AoA-like structures and other heterogeneous positional data.

### Represent Numeric Properties Through Stringified Keys

Another possibility would be to model numeric indices as string properties:

```luau
type T = {
    ["1"]: number,
    ["2"]: string,
}
```

This does not accurately represent Lua table semantics because numeric keys and string keys are distinct runtime keys.

For example:

```luau
local value = {
    [1] = 10,
    ["1"] = "hello",
}
```

contains two different properties.

Using string literals to represent numeric keys would therefore introduce an incorrect correspondence between the type system and runtime behavior.

### Introduce Separate Syntax for Numeric Properties

A dedicated syntax could theoretically be introduced for numeric concrete properties.

For example:

```luau
type T = {
    property[1]: number,
}
```

However, this would introduce additional syntax for a concept that can already be represented naturally using bracketed literal keys:

```luau
type T = {
    [1]: number,
}
```

There is no significant semantic advantage to introducing a separate construct.

### Treat Numeric Literals as Indexers

Another option would be to continue interpreting:

```luau
type T = {
    [1]: number,
}
```

as an indexer.

This would preserve a uniform syntactic interpretation of bracketed keys, but it would prevent the type system from distinguishing a specific numeric property from an arbitrary numeric index.

Such a model cannot accurately express heterogeneous numeric table shapes and would unnecessarily discard information that is already available statically.

## Future Extensions

Once numeric singleton types can represent concrete properties, the mechanism can serve as a foundation for additional type-system features.

In particular, it can support more precise representations of heterogeneous arrays and positional data:

```luau
type T = {
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

It can also provide a basis for future integration between type packs and heterogeneous table types, where each element of a type pack corresponds to a concrete numeric property.

The general model becomes:

```text
singleton key
    -> concrete property

non-singleton key
    -> indexer
```

This model is consistent with the treatment of concrete string properties while extending the same concept to numeric keys.

## Conclusion

Numeric literals are already meaningful runtime table keys, and numeric singleton types provide a natural type-level representation of those exact values.

The proposed change allows the type system to distinguish:

```luau
type T = {
    [1]: number,
    [2]: string,
}
```

from:

```luau
type T = {
    [number]: number,
}
```

The former describes two specific properties with different types, while the latter describes an arbitrary set of numeric keys sharing one value type.

Numeric singleton aliases would behave equivalently:

```luau
type Index = 1

type T = {
    [Index]: number,
}
```

would be equivalent to:

```luau
type T = {
    [1]: number,
}
```

This extends the existing concrete-property model to numeric keys without requiring runtime changes or introducing new syntax, while providing the type-system foundation needed to describe heterogeneous numeric structures and, potentially, type-pack-backed table layouts.
