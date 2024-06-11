# `rawget` type operator

## Summary

This RFC proposes the addition on one type operator, `rawget`, which can be used to look up a specific property of another type *without* invoking the `__index` metamethod.

## Motivation

Given that `rawget` is a built-in runtime operator in the language ([rawget Lua Globals](https://create.roblox.com/docs/reference/engine/globals/LuaGlobals#rawget)), it feels natural for there to be a version of this as a type operator. As such, the main motivation behind this feature is to close out holes in Luau's type families and allow Luau developers to be more expressive writing typed code:

```luau
local prop: rawget<someTy, someProp> = rawget(someTy, someProp)
```

## Design

The functionality of `rawget` type operator behaves the same as its [runtime pair](https://create.roblox.com/docs/reference/engine/globals/LuaGlobals#rawget): provide a way to look up a specific property of a type without invoking the `__index` metamethod. 
 
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

The implementation effort for this type operator is very minimal. Since the `rawget` type operator functions similarly to the `index` type operator, we can reuse the functions already used to implement the `index` type operator.

## Drawbacks

There aren't appreciable drawbacks. One possible drawback is the increase in the size of the codebase, which may complicate maintenance. However, since code for `index` type operators already exists, the additional code needed to implement this operator will be minimal.

In summary, the implementation of this feature will be relatively straightforward while minimally increasing the lines of code. This approach ensures maintainability and leverages existing code, reducing potential errors and simplifying future updates. Therefore, the benefits of adding this operator outweigh the minor increase in code size, making it a practical and efficient enhancement.

## Alternatives

An alternative to the `rawget` type operator is to enable users to modify the behavior of the existing `index` type operator to control its recursive nature. While this approach offers flexibility, it requires additional design considerations and complicates both usage and implementation. However, the introduction of "user-defined type functions" is planned for the near future. This will allow users to create their own type operators, making this alternative design feasible for developers to implement.