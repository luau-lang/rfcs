# Expanding Type Packs into Table Properties

## Dependencies

1. [Singleton Type Literals as Table Property Keys](singleton-indexer-to-property)
2. [Numeric Type Literals as Concrete Table Property Keys](number-literals)

## Summary

Allow type packs to be automatically expanded into concrete numeric table properties, with one property generated for each type contained in the pack.

For a type pack:

```luau
type A<T...> = {
    B: {T...},
}
```

the proposed semantics are equivalent to expanding the pack into individually typed numeric properties:

```luau
type A<T...> = {
    B: {
        [1]: T1,
        [2]: T2,
        [3]: T3,
        -- etc.
    }
}
```

where the number of properties is determined by the number of types in the pack.

This provides a type-level representation for heterogeneous, statically sized tables and allows the type information contained in a type pack to be preserved when values are stored in tables.

The proposal is primarily intended to make heterogeneous array-like structures expressible without reducing them to a common numeric indexer type.

## Motivation

This proposal depends on the ability to represent concrete numeric table properties through numeric singleton types.

The second dependency establishes the distinction between a numeric singleton and a numeric indexer:

```luau
[1]: T
```

represents a concrete property at index `1`, while:

```luau
[number]: T
```

represents an arbitrary numeric index.

This distinction is necessary because a type pack contains a finite, ordered sequence of types. It therefore contains more information than a numeric indexer can represent.

Consider a type pack:

```text
number, string, boolean
```

A numeric indexer could only represent this as something similar to:

```luau
[number]: number | string | boolean
```

This loses the correspondence between an index and its type.

The desired representation is:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

This preserves the positional information:

```text
1 -> number
2 -> string
3 -> boolean
```

The distinction becomes particularly important for functions such as `unpack`.

Conceptually, `unpack` can be described as a function whose result is a type pack:

```luau
unpack: <T, U>(that: {[T]: U}) -> U...
```

The resulting type pack can then be assigned to multiple local variables:

```luau
local a0, a1, a2 = unpack(target_table)
```

When the type checker knows that the result pack contains exactly three types, it can preserve the correspondence between those returned values and their destinations. Excess destinations can also be handled according to the existing type-pack semantics.

The problem is that the reverse operation is currently difficult to express.

A type pack can describe:

```text
T1, T2, T3, ...
```

but a table type cannot naturally express that the first table element has type `T1`, the second has type `T2`, the third has type `T3`, and so on.

As a result, code that stores a heterogeneous type pack in a table has to choose between losing type information or introducing an additional abstraction.

For example, a structure may need to be represented approximately as:

```luau
type Package<T... = ...unknown> = {
    read: {unknown},
    readUnpack: (self: Package<T...>) -> T...,
}
```

The `read` field cannot preserve the actual types in `T...`. It has to use an indexer such as `{unknown}` because the type pack cannot directly describe the table's individual elements.

The proposed feature removes this limitation by allowing the type pack itself to define the concrete numeric properties of the table.

## Core Concept

A type pack represents an ordered sequence of types:

```text
T...
=
T1, T2, T3, ..., Tn
```

When such a pack appears as the element type of a table in a supported position:

```luau
{T...}
```

the pack should be expanded into concrete numeric properties:

```luau
{
    [1]: T1,
    [2]: T2,
    [3]: T3,
    ...
    [n]: Tn,
}
```

The important distinction is that this is not an indexer.

It does not mean:

```luau
[number]: T1 | T2 | T3
```

Instead, each position receives its own property and therefore retains its own type.

Conceptually:

```text
type pack
    ↓
ordered sequence
    ↓
numeric properties
    ↓
concrete heterogeneous table
```

## Design

### Basic Expansion

Given:

```luau
type A<T...> = {
    B: {T...},
}
```

and an instantiation:

```luau
type X = A<number, string, boolean>
```

the field `B` is equivalent to:

```luau
B: {
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

Therefore:

```luau
local value: X
```

has the following statically known structure:

```text
value.B[1] -> number
value.B[2] -> string
value.B[3] -> boolean
```

The number of concrete properties is determined by the number of elements in the instantiated type pack.

### Empty Packs

An empty type pack produces an empty positional structure.

For example:

```luau
type A<T...> = {
    B: {T...},
}

type X = A<>
```

produces a table with no concrete numeric properties:

```luau
type X = {
    B: {},
}
```

No numeric indexer is implicitly introduced.

This is important because an empty pack does not mean "an arbitrary number of values". It means that there are zero values.

### Single-Element Packs

A single-element pack:

```luau
type X = A<number>
```

produces:

```luau
type X = {
    B: {
        [1]: number,
    },
}
```

This remains a concrete property rather than becoming:

```luau
[number]: number
```

The number of elements in the pack is part of its type-level information and must not be discarded.

### Nested Packs

The same mechanism can be applied recursively to nested table types.

For example:

```luau
type A<T...> = {
    B: {
        C: {T...},
    },
}
```

with:

```luau
type X = A<number, string>
```

produces:

```luau
type X = {
    B: {
        C: {
            [1]: number,
            [2]: string,
        },
    },
}
```

The expansion is therefore structural and does not require a separate representation for each nesting level.

## Heterogeneous Arrays

One of the primary use cases is representing heterogeneous array-like structures.

Consider:

```luau
local value = {
    42,
    "hello",
    true,
}
```

A homogeneous array representation:

```luau
{number | string | boolean}
```

cannot preserve the fact that each position has a different type.

With type-pack expansion, the structure can instead be represented as:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

This means that accesses retain their exact types:

```luau
value[1] -- number
value[2] -- string
value[3] -- boolean
```

This is effectively a heterogeneous tuple represented using ordinary Lua table indexing.

## Array of Arrays

A more important use case is dynamically constructed Array-of-Arrays structures.

For example, suppose a generic structure stores rows whose types are determined by a type pack:

```luau
type Row<T...> = {
    [1]: T1,
    [2]: T2,
    [3]: T3,
}
```

With pack expansion, the representation can be generated automatically:

```luau
type Row<T...> = {T...}
```

and:

```luau
type Row<number, string, boolean>
```

becomes:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

An outer table can then contain multiple such rows without manually defining every possible row shape.

This makes structures such as:

```text
[
    [number, string, boolean],
    [number, string, boolean],
    [number, string, boolean]
]
```

representable while preserving the type of each position.

The important property is that the structure remains statically heterogeneous rather than collapsing every value to a common union.

## Interaction with `unpack`

The proposal is particularly useful when combined with functions that convert table elements into type packs.

Conceptually:

```luau
unpack: <T, U>(that: {[T]: U}) -> U...
```

takes a table and produces a type pack.

With a concrete heterogeneous table:

```luau
type T = {
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

`unpack` can conceptually produce:

```text
number, string, boolean
```

which can then be assigned positionally:

```luau
local a, b, c = unpack(value)
```

with:

```text
a -> number
b -> string
c -> boolean
```

The reverse direction is what this proposal makes expressible:

```text
T1, T2, T3
    ↓
{
    [1]: T1,
    [2]: T2,
    [3]: T3,
}
```

Together, these operations provide a natural correspondence between type packs and heterogeneous numeric tables:

```text
table
  ↓ unpack
type pack
  ↓ expansion
table
```

This does not necessarily imply that these operations are runtime inverses. The proposal concerns the type-level representation and preservation of positional information.

## Known Length

A critical property of this design is that the table's known length is derived from the type pack.

For:

```luau
type T<A...> = {A...}
```

the number of concrete numeric properties is exactly the number of elements in `A...`.

For example:

```luau
type T = <number, string, boolean>
```

corresponds to:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

This is different from:

```luau
{
    [number]: number | string | boolean,
}
```

because the latter has no finite statically known length.

The distinction can therefore be expressed as:

```text
type pack
    -> finite known sequence
    -> concrete numeric properties

numeric indexer
    -> arbitrary sequence
    -> unknown number of elements
```

This allows the indexer to retain its original role: describing tables where the exact set or count of keys cannot be determined statically.

## Indexers Remain Unchanged

The proposal does not remove or redefine numeric indexers.

For example:

```luau
type T = {
    [number]: string,
}
```

continues to describe an arbitrary number of numeric properties.

Likewise:

```luau
type T = {string}
```

continues to describe a homogeneous array-like table.

The distinction is therefore:

```text
{string}
    -> arbitrary number of numeric elements
    -> every element is string

{string, number, boolean}
    -> finite known sequence
    -> each position has its own type
```

The latter is the capability provided by type-pack expansion.

## Concrete Properties and Indexers

Concrete properties and indexers may coexist.

For example:

```luau
type T<T...> = {
    [1]: string,
    [T...],
    [number]: unknown,
}
```

The exact syntax and precedence rules for mixing a pack expansion with explicit properties and indexers would need to follow the existing table-property model.

The general semantic principle should be that concrete properties retain their specific types while the indexer describes keys that are not represented by a concrete property.

For example, conceptually:

```luau
type T = {
    [1]: number,
    [2]: string,
    [number]: unknown,
}
```

should preserve:

```text
T[1] -> number
T[2] -> string
T[other number] -> unknown
```

This follows the same property-over-indexer precedence described by the singleton-key proposals.

## Type Pack Aliases

The pack does not need to be written directly at the point where the table type is declared.

For example:

```luau
type Row<T...> = {T...}

type Table<T...> = {
    rows: Row<T...>,
}
```

Given:

```luau
type X = Table<number, string, boolean>
```

the resulting shape is:

```luau
type X = {
    rows: {
        [1]: number,
        [2]: string,
        [3]: boolean,
    },
}
```

This makes type packs composable through generic type aliases rather than requiring the final table shape to be written manually.

## Runtime Semantics

No runtime changes are required.

The expansion exists exclusively in the type system.

For example:

```luau
type Row<T...> = {T...}

type X = Row<number, string, boolean>
```

does not create a special runtime table representation.

The runtime value is still an ordinary Lua table:

```luau
local row = {
    42,
    "hello",
    true,
}
```

The type checker simply understands the statically known shape as:

```text
1 -> number
2 -> string
3 -> boolean
```

No metadata about the type pack needs to exist at runtime.

## Type Safety

The primary benefit is preserving positional type information.

Given:

```luau
type Row = {
    [1]: number,
    [2]: string,
}
```

the following should be rejected:

```luau
local row: Row = {
    "wrong",
    123,
}
```

because:

```text
row[1] requires number
row[2] requires string
```

Likewise, accesses retain their precise types:

```luau
local n = row[1] -- number
local s = row[2] -- string
```

This information would otherwise be lost if the table were represented using a union-valued numeric indexer.

## Bounds and Unknown Indices

A concrete numeric property only describes a statically known key.

For:

```luau
type T = {
    [1]: number,
    [2]: string,
}
```

the type checker knows about indices `1` and `2`.

An access using a statically unknown numeric value cannot be assumed to refer to either property unless an appropriate indexer is present.

For example:

```luau
local index: number = ...
value[index]
```

cannot in general be treated as either `number` or `string`, because `index` may refer to an arbitrary numeric key.

This is another reason not to model the expanded pack as a numeric indexer.

The exact diagnostic and resulting type for out-of-range or unknown numeric accesses should follow the existing table-indexing rules.

## Variadic Generic Functions

The feature also provides a natural way to preserve relationships between variadic generic parameters and stored table values.

Consider a conceptual container:

```luau
type Package<T...> = {
    data: {T...},
    unpack: (self: Package<T...>) -> T...,
}
```

For:

```luau
type P = Package<number, string, boolean>
```

the `data` field becomes:

```luau
{
    [1]: number,
    [2]: string,
    [3]: boolean,
}
```

while `unpack` returns:

```text
number, string, boolean
```

The same type pack therefore describes both sides of the relationship:

```text
data:
    1 -> number
    2 -> string
    3 -> boolean

unpack():
    number, string, boolean
```

This removes the need to weaken `data` to `{unknown}` merely because the table type system cannot currently express the contents of a pack.

## Drawbacks

The main drawback is increased complexity in the type checker.

A table type containing a type-pack expansion cannot be fully represented until the pack has been resolved. The checker therefore needs to preserve and expand pack information as part of table-type construction.

This becomes more complicated when packs interact with:

* generic parameters;
* unresolved type packs;
* nested generic aliases;
* unions;
* intersections;
* table indexers;
* explicit numeric properties;
* recursive types.

Another consideration is that table types may become significantly larger after expansion.

For example, a pack containing hundreds of types would result in hundreds of concrete properties:

```luau
{
    [1]: T1,
    [2]: T2,
    ...
    [100]: T100,
}
```

The implementation may therefore need an internal representation that can preserve pack structure without eagerly materializing every property in every situation.

A further issue is interaction with operations that modify tables dynamically.

Lua tables are mutable and do not inherently have a fixed length. A type-level representation derived from a finite type pack therefore describes the statically known portion of the table rather than imposing a runtime restriction that the table must physically contain exactly those keys.

The type system must consequently continue to distinguish between:

```text
statically known properties
```

and:

```text
runtime table contents
```

The feature should not imply that a table represented by a finite pack is immutable or that no additional keys can exist unless existing Luau table semantics already require such a restriction.

## Alternatives

### Continue Using a Numeric Indexer

The simplest alternative is to represent all table contents using:

```luau
[number]: U
```

or:

```luau
{U}
```

This works well for homogeneous arrays.

It does not work for heterogeneous structures because the indexer cannot preserve the relationship between a particular index and its type.

For example:

```luau
[number]: number | string | boolean
```

cannot distinguish:

```text
1 -> number
2 -> string
3 -> boolean
```

from any other arrangement of those types.

### Store the Data as `unknown`

A generic container could instead use:

```luau
type Package<T...> = {
    data: {unknown},
    unpack: (self: Package<T...>) -> T...,
}
```

This preserves the return type of `unpack`, but loses type information when accessing `data` directly.

The user must then rely on the unpacking operation or perform additional type narrowing.

This effectively creates two separate type representations for the same data:

```text
data -> unknown
unpack(data) -> T...
```

The proposed expansion makes the relationship explicit in the table type itself.

### Manually Generate Concrete Properties

Users could manually write:

```luau
type T = {
    [1]: T1,
    [2]: T2,
    [3]: T3,
}
```

This works for fixed structures but cannot naturally express a generic number of elements determined by a type pack.

A generic abstraction would have to manually enumerate a maximum number of supported positions, which is neither scalable nor type-safe for arbitrary packs.

### Introduce a Dedicated Tuple Type

Another alternative would be to introduce a separate tuple type construct rather than representing heterogeneous positional data using ordinary tables.

For example, a hypothetical:

```luau
tuple<T...>
```

could represent:

```text
T1, T2, T3, ...
```

This could provide a clean abstraction for positional data, but it would introduce a separate type representation for a structure that is already naturally represented by Lua tables.

It would also require defining how tuples interact with normal table operations, indexing, mutation, metatables, and existing table types.

The proposed design instead reuses the existing table property system.

### Special Syntax for Pack Expansion

Another option would be to introduce explicit syntax:

```luau
type T<T...> = {
    [...T]: ...
}
```

or another dedicated construct indicating that a type pack should be expanded.

The disadvantage is that this introduces new syntax for an operation that can be naturally inferred from the occurrence of a type pack in a table element position.

If the language already understands that `{T...}` represents the application of a type pack to a table, implicit expansion keeps the feature smaller and more composable.

## Future Extensions

Type-pack expansion into concrete numeric properties could serve as a foundation for more advanced relationships between type packs and table types.

Potential future applications include:

* heterogeneous arrays;
* tuple-like tables;
* Array-of-Arrays structures;
* generic row/column data structures;
* strongly typed serialization formats;
* generic containers that preserve positional types;
* improved typing of `pack`/`unpack`-style operations;
* compile-time transformations between type packs and table shapes.

The same principle could potentially be generalized beyond numeric table properties if the language gains other forms of statically enumerable table keys.

The underlying model remains:

```text
finite type pack
    ↓
ordered sequence of types
    ↓
concrete numeric properties
```

while:

```text
non-singleton numeric key
    ↓
numeric indexer
```

continues to represent an unbounded or statically unknown set of numeric keys.

## Conclusion

Type packs already provide a representation for a finite, ordered sequence of types:

```text
T1, T2, T3, ...
```

Tables already provide a representation for concrete properties:

```luau
{
    [1]: T1,
    [2]: T2,
    [3]: T3,
}
```

The missing operation is the ability to connect these two representations.

This proposal introduces that connection by expanding a type pack into concrete numeric properties:

```luau
type A<T...> = {
    B: {T...},
}
```

becomes, for an instantiated pack:

```luau
type A<number, string, boolean> = {
    B: {
        [1]: number,
        [2]: string,
        [3]: boolean,
    },
}
```

The result preserves both the number of values and the type associated with each position.

This is fundamentally different from using a numeric indexer:

```luau
[number]: number | string | boolean
```

because the indexer describes arbitrary numeric keys while the expanded pack describes a finite set of concrete properties.

Combined with the proposed singleton and numeric-literal property semantics, this provides a coherent model:

```text
singleton key
    -> concrete property

non-singleton key
    -> indexer

finite type pack
    -> ordered set of concrete numeric properties
```

The result is a more expressive table type system that can represent heterogeneous positional structures without weakening their types to a common indexer type, while requiring no changes to the runtime representation of Lua tables.
