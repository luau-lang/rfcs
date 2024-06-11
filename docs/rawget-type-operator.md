# `rawget` type operator

## Summary

This RFC proposes the addition on one type operator, `rawget`, which can be used to look up a specific property of another type *without* invoking the `__index` metamethod.

## Motivation

There exists an `index` type operator that allows developers to obtain a type of a property from classes / tables. If a type is not found in the given class / table, the operator recursively indexes into the `__index` metamethod to continue searching for the property. Sometimes, this could be an unwanted behavior. For example, given this code: 

```luau
local var1 = {
  property = "hello"
}

local var2 = setmetatable({ }, { __index = seq })

type doesExist = index<typeof(var2), "property"> -- reduces to `string`
```
The `index` type operator is used to obtain the type of `property` from a class `var1`. The operator continues to search recursively through the `__index` metamethod if the property is not initially found. This behavior can lead to unexpected results, as shown in the code snippet where the `__index` metamethod of `var2` points to `var1`, causing the `index` operator to return `string` for the `property` key even though `var2` itself does not have a property.

This can be undesirable for developers who expect the `index` operator to only check the properties directly on the variable without considering the metamethod chain. In the given example, the expectation might be to receive an error when querying `var2` for `property`, indicating that the property does not exist on `var2`.

To address this issue, we want to reconsider the behavior of the `index` type operator in the context of the `__index` metamethod. Developers may require a more predictable and controlled mechanism for type checking that does not involve recursively following the `__index` chain.

## Design

The proposed solution is the implementation of `rawget` type operator. Like the existing [rawget Lua Globals](https://create.roblox.com/docs/reference/engine/globals/LuaGlobals#rawget), this type operator aims to provide a way to look up a specific property of a type without invoking the `__index` metamethod. 
 
```luau
type doesExist = rawget<typeof(var2), "property"> -- reduces to an error
```

Error messages would be consistent with those of the `index` type operator:
```luau
type Person = {
  age: number,
  name: string,
  alive: boolean
}

type age = rawget<Person, "ager"> -- Error message: Property '"ager"' does not exist on type 'Person'

local key = "age"
type age = rawget<Person, key> -- Error message: Second argument to rawget<Person,_> is not a valid index type; Unknown type 'key'
```

The implementation effort for this type operator is relatively small. Since the `rawget` type operator functions similarly to the `index` type operator, we can reuse the functions already used to implement the `index` type operator.

## Drawbacks

There aren't appreciable drawbacks. One possible drawback is the increase in the size of the codebase, which may complicate maintenance. However, since code for `index` type operators already exists, the additional code needed to implement this operator will be minimal.

In summary, the implementation of this feature will be relatively straightforward while minimally increasing the lines of code. This approach ensures maintainability and leverages existing code, reducing potential errors and simplifying future updates. Therefore, the benefits of adding this operator outweigh the minor increase in code size, making it a practical and efficient enhancement.

## Alternatives

An alternative to the `rawget` type operator is to enable users to modify the behavior of the existing `index` type operator to control its recursive nature. While this approach offers flexibility, it requires additional design considerations and complicates both usage and implementation. However, the introduction of "user-defined type functions" is planned for the near future. This will allow users to create their own type operators, making this alternative design feasible for developers to implement.