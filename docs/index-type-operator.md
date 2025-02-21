# `index` type function

**Status**: Implemented

## Summary

This RFC proposes the addition of one type function, `index`, which can be used to look up a specific property of another type (like TypeScript's Indexed Access Type).

## Motivation

The primary motivation of this proposal is to allow Luau to define a type by accessing a property of another existing type. For instance, consider the following example code:
```luau
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

This is a valid Luau program; however, in order to define the type of `Person["age"]` we had to first declare a variable `bob` and utilize the `typeof` type function. This is quite cumbersome when developers want to typecheck using the type of `Person["age"]` without having to declare a variable first. Additionally, in order to define the union type of all the properties of `Person`, current Luau requires an explicit list of each property using `typeof`.

The expected outcome of the index type function is that it will enhance developer experience and allow Luau developers to more easily develop well-typed programs.

## Design

The solution to this problem is a type function, `index`, that can compute the type based on the static properties of `Person`. Formally, the `index` type function will take in two arguments: the type to index (indexee) and the type to index it with (indexer). This would allow us to instead write the following code:
```luau
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

Error messages will be displayed for incorrect type arguments. If the indexer is not a property in the indexee, 
```luau
type age = index<Person, "ager"> -- Error message: Property '"ager"' does not exist on type 'Person'
```
If the indexer is not a type,
```luau
local key = "age"
type age = index<Person, key> -- Error message: Second argument to index<Person,_> is not a valid index type; Unknown type 'key'
```
Note: these errors will be part of the general type function reduction errors since `index` will be built into the type function system.

The indexee may be a union type. In this case, the type function will distribute the arguments to multiple type families:
```luau
type Person2 = {
  age: string
}

-- equivalent of `index<Person, "age"> | index<Person2, "age">`
type idxType3 = index<Person | Person2, "age"> -- idxType3 = number | string

-- equivalent of `index<Person, "alive" | "age"> | index<Person2, "alive" | "age">`
type idxType4 = index<Person | Person2, "alive" | "age"> -- Error message: Property '"age" | "alive"' does not exist on type 'Person | Person2'
```

In the circumstance that the indexee's type is a class or table with an `__index` metamethod, the `__index` metamethod will *only* be invoked if the indexer is not found within the current scope:
```luau
local exampleClass = { Foo = "eight" }
local exampleClass2 = setmetatable({ Foo = 8 }, { __index = exampleClass })
local exampleClass3 = setmetatable({ Bar = "nine" }, { __index = exampleClass })

type exampleTy2 = index<typeof(exampleClass2), "Foo"> -- exampleTy2 = number
type exampleTy3 = index<typeof(exampleClass3), "Foo"> -- exampleTy3 = string
```

One edge case to consider when using/designing this type function is that `__index` only supports 100 nested `__index` metamethods until it gives up. In the case that a property is not found within the 100 recursive calls, this type function will fail to reduce.

Implementation is straight forward: the type of the indexee will be determined (table, class, etc) -> search through the properties of the indexee and reduce to the corresponding type of the indexer if it exists; otherwise, reduce to an error. 

## Drawbacks

A drawback to this feature is the possible increase in the cost of maintenance. In the end, this RFC proposes adding another built-in type functions to the new type system. However, the addition of this feature may be worthwhile, as the `index` type function is a useful type feature that:
1. Alleviates the need to manually keep types in sync
2. Provides a powerful way to access the properties of an object and perform various operations on it with other type functions
3. Allows the community to write code with fewer errors and more safety

## Alternatives

An alternative design can be depicted from the example below:
```luau
type Person3 = {
  age: number,
  name: string,
  alive: boolean,
  job: string
}

local function edgeCase(p: Person)
  type unknownType = index<typeof(p), "job">
end
```
In our current design, the program simply fails to reduce (and throws an error). However, it is worth noting that `index<typeof(p), "job">` can also reduce to type `unknown` because the parameter `p` can be of type `Person` or `Person3` (since tables support width subtyping; hence, `Person3` is a subtype of `Person`):
- If `p` is of type `Person`, `index<typeof(p), "job">` should reduce to an error. 
- If `p` is of type `Person3`, `index<typeof(p), "job">` should reduce to type `string`.

Because there are conflicting types for `p` depending on the run time, it is safest for the program to reduce to a type `unknown`. In this design, we would need a way to determine if the indexee can be different at runtime. We could determine this through the implementation of more table types, specifically exact and inexact table types. Then, the program will have two cases when an indexee does not contain the indexer type:
1. If the indexee is an inexact table type, reduce to an `unknown` type.
2. If the indexee is an exact table type, fail to reduce and throw an error.

FYI: exact table type indicates that the table has only the properties listed in its type, and inexact table type indicates that the table has at least the properties listed in its type.

Later down the line, we can also consider adding syntactic sugar for this type function. Instead of doing:
```luau
type name = index<Person, "name">
```
We could use:
```luau
type name = Person["name"]
```
or 
```luau
type name = Person.name
```
or even both!