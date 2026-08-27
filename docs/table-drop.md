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
`table.remove` defaults to removing the last element of a table when its second argument is either missing or `nil`. This creates the footgun of unintentionally removing the last element of the table unless you use an `if` statement.

## Design

First, I am proposing a standard library function for tables to remove elements by value rather than index:

```table.drop(t: {V}, value: V, max: number?)```

`t` refers to the table you're using, `value` is the value you want removed, and `max` is an optional argument specifying the limit of how many occurrences of `value` you want removed.. If `max` is `nil`, every occurrence of `value` is removed from the table. This function does not return anything due to the existence of `table.remove`.

This function traverses front-to-back (index 1 to `#t`) to find indices where the table element at that index is equal to `value` (using `__eq` semantics), breaking early if `max` is specified and `max` elements have already been found. The indices of elements to remove are pushed in a last-in first-out manner (adding later matches in the table closer to the front of the stack), so that we can directly iterate the stack to remove indices back-to-front so that elements are removed in the correct order. This ensures that we remove the correct indices without worrying about shifting during array removal.

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
table.remove(t, table.find("banana")) -- Warning: Using optional variables as the 2nd arg for table.remove is a footgun. Use table.drop instead.
```

## Drawbacks

While there are no compatibility concerns, this does add a new standard library function to the language.

## Alternatives

- Simply adding the linter rule would stop the footgun, but it would leave the verbose use of `if` statements as the only solution.
- Make the linter rule only apply to uses of `table.remove` where function calls that return `number?` are used for the 2nd argument
- Have `table.drop` return how many times it removed `value`
- Renaming `table.drop` (`table.erase`, `table.remval`, etc)
