# `index` type operator

## Summary

This RFC proposes the addition of one type operator, `index`, which can be used to look up a specific property of another type (like TypeScript's Indexed Access Type).

## Motivation

The primary motivation of this proposal is to allow Luau to define a type by accessing a property of another existing type. For instance, consider the following example code:
```lua
type Person = {
  age: number,
  name: string,
  alive: boolean
}

local bob: Person = {
  age = 22,
  name = "Bob",
  alive = true
}

local function doSmt(param: typeof(bob["age"])) -- param = number
  -- rest of code
end

type unionType = typeof(bob["age"]) | typeof(bob["name"]) | typeof(bob["alive"]) -- unionType = number | string | boolean
```

This is a valid Luau program; however, in order to define the type of `Person["age"]` we had to first declare a variable `bob` and utilize the `typeof` type operator. This is quite cumbersome when developers want to typecheck using the type of `Person["age"]` without having to declare a variable first. Additionally, in order to define the union type of all the properties of `Person`, current Luau requires an explicit list of each property using `typeof`.

The expected outcome of the index type operator is that it will enhance developer experience and allow Luau developers to more easily develop well-typed programs.

## Design

The solution to this problem is a type operator, `index`, that can compute the type based on the static properties of `Person`. Formally, the `index` type operator will take in two arguments: the type to index and the type to index it with. This would allow us to instead write the following code:
```lua
type Person = {
  age: number,
  name: string,
  alive: boolean
}

local function doSmt(param: index<Person, "age">) -- param = number
  -- rest of code
end

type idxType = index<Person, keyof<Person>> -- idxType = number | string | boolean

type idxType2 = index<Person, "age" | "name"> -- idxType2 = number | string
```

Now, the type of `doSmt()`'s parameter can be defined without declaring a variable `bob`. Additionally, regardless of how the type `Person` grows, `idxType` will always be defined as the union of all the properties.

Error messages will be displayed for incorrect syntax. If the given type used to access a property is invalid, 
```lua
type age = index<Person, "ager"> -- Error message: Property 'ager' does not exist on type 'Person'.
```
If the given value used to access a property is not a type,
```lua
local key = "age"
type age = index<Person, key> -- Error message: Type 'key' cannot be used as an index type.
```
Note: these errors will be part of the general type family reduction errors since `index` will be built into the type family system.

Implementation is straight forward: the type of the indexee will be determined (table, array, etc) -> search through the properties of the indexee and return the corresponding type of the indexer if it exists; otherwise, return an error or unknown type depending on the scope. 

## Drawbacks

A drawback to this feature is the possible increase in cost of maintenance. In the end, this RFC proposes adding another built-in type operators to the new type system. However, the addition of this feature may be worthwhile as the `index` type operator is a useful type feature that:
1. Alleviates the need to manually keep types in sync
2. Provides powerful way to access the properties of an object and perform various operations on them with other type operators
2. And ultimately, allows the community to write code with fewer errors and more safety

## Alternatives

An alternative to the proposed implementation is utilizing inexact table types and exact table types (to be implemented) to handle the edge in a simpler manner. If we know that the indexee is an inexact table type, the index type operator can return the `unknown` type. If we know that the indexee is an exact table type, we can return an error.
