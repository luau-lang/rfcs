# `rawget` type operator

## Summary

This RFC proposes the addition of a new type operator, `rawget`, which can be used to look up a specific property of a table type *without* invoking the `__index` metamethod.

## Motivation

Currently, `rawget` is a built-in runtime operator in the language ([rawget in Luau Global functions](https://luau-lang.org/library#global-functions)) that is missing a corresponding type operator that captures its behavior. This RFC seeks to address this hole in the type system by providing a builtin type operator for `rawget` that will allow Luau developers to express more accurate types in their code:

```luau
local prop: rawget<someTy, someProp> = rawget(someTy, someProp)
```

## Design

The functionality of `rawget` type operator behaves the same as its [runtime counterpart](https://luau-lang.org/library#global-functions): provide a way to look up a specific property of a table type without invoking the `__index` metamethod. 
 
```luau
local var1 = {
  property = "Hello"
}

local var2 = setmetatable({ }, { __index = var1 })
type doesntExist = rawget<typeof(var2), "property"> -- reduces to an error

local var3 = setmetatable({ property = 1 }, { __index = var1 })
type doesExist = rawget<typeof(var3), "property"> -- doesExist = number
```

Error messages would be consistent with those of the `index` type operator:
```luau
type doesntExist = rawget<typeof(var2), "property">  -- Error message: Property '"property"' does not exist on type 'var2'

local key = "property"
type age = rawget<Person, key> -- Error message: Second argument to rawget<Person, _> is not a valid index type; Unknown type 'key'
```

Note: `rawget` type operator does not work on class types because they do not have direct fields that can be looked up without invoking its metamethod.

The implementation effort for this type operator is very minimal. Since the `rawget` type operator functions similarly to the `index` type operator, we can reuse the functions already used to implement the `index` type operator.

## Drawbacks

There aren't appreciable drawbacks. One possible drawback is the increase in the size of the codebase, which may complicate maintenance. However, this is not expected to be a major drawback since the code for `index` type operators already exists. By reusing the code, it is even fair to say that there will be less than 50 lines of code needed to implement this type operator.

Another drawback can come from the extra knowledge a user may need in order to use this type operator. For example, users will need to know the inner workings of types and how different operations interact with them (for instance, how `index` interacts with metatables that have the `__index` metamethod). However, there are documentations that outline each interaction, and thus this drawback does not seem to pose an issue.

## Alternatives

An alternative to the `rawget` type operator is to enable users to modify the behavior of the existing `index` type operator to control its recursive nature. While this approach offers flexibility, it requires additional design considerations and complicates both usage and implementation. However, the introduction of "user-defined type functions" is planned for the near future. This will allow users to create their own type operators, making this alternative design feasible for developers to implement.