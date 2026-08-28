# `table.drop` & `table.remove` lint improvement

## Summary

Add the `table.drop` standard library method for removing table elements by value rather than index, along with a linter improvement
to prevent the `table.remove` footgun of accidental tail amputation.

## Motivation

To remove table elements by value rather than index, the following pattern is commonly used:

```luau
table.remove(t, table.find(t, element))
```

However, `table.find` returns `nil` when it cannot find the element specified in the second argument of the function and
`table.remove` defaults to removing the last element of a table when its second argument is either missing or `nil`. This creates the footgun of unintentionally removing the last element of the table unless you use an `if` statement or the global `assert` function.

## Design

First, I am proposing a standard library function for tables to remove elements by value rather than index:

```table.drop(t: {V}, value: V, count: number?)```

`t` refers to the table you're using, `value` is the value you want removed, and `count` is an optional argument specifying the limit of how many occurrences of `value` you want removed. If `count` is `nil`, every occurrence of `value` is removed from the table. This function does not return anything.

This function traverses front-to-back (index 1 to `#t`) to find indices where the table element at that index is equal to `value` (using `__eq` semantics), breaking early if `count` is specified and `count` elements have already been found.

For example:
```luau
local t = {"apple", "banana", "orange", "orange"}

table.drop(t, "orange") -- t is now {"apple", "banana"}
table.insert(t, "banana") -- t is now {"apple", "banana", "banana"}
table.drop(t, "banana", 1) -- t is now {"apple", "banana"}
```

Next, I am proposing to add a linter rule warning the user about the `table.remove` footgun if their second argument is an optional variable and isn't explicitly `nil`.

```luau
local t = {"apple", "banana", "orange"}

table.remove(t, nil) -- OK
table.remove(t, table.find(t, "banana")) -- Warning: Using optional variables as the 2nd arg for table.remove is a footgun. Use table.drop instead.
```

## Drawbacks

While there are no compatibility concerns, this does add a new standard library function to the language.

## Alternatives

- Simply adding the linter rule would help prevent the `table.remove` footgun, but it would leave `if` statements and `assert` calls as the only solutions, both of which being fairly verbose.
- Make the linter rule only apply to uses of `table.remove` where function calls that return `number?` are used for the 2nd argument.
- Have `table.drop` return how many times it removed `value`.
- Renaming `table.drop` (`table.erase`, `table.remval`, etc).
- Adding optional `start` and `finish` arguments to `table.drop` to specify search distance within the table.
