# Named Function Expressions

## Summary

This proposes that the current function expressions, which are always anonymous (unnamed) be allowed to have names.

## Motivation

This would make some code more readable, enable Luau to give function expressions names other than `<anonymous>` in stack traces and with `debug.info`, enable function expressions to be recursive (call themselves), and help in miscellaneous profiling and debugging tools.

## Design

The current grammar for function expressions is the following, taken from the `simpleexp` rule.

```
attributes 'function' funcbody
```

This proposal would change that grammar to the following.

```
attributes 'function' [NAME] funcbody
```

This adds an optional name in function expressions, which can be used like so:

```luau
foo(function factorial(n)
	return if n == 1 then 1 else n * factorial(n - 1)
end)
```

It is also important to note where this creates a binding to the function expression, the following example should explain:

```luau
-- 'bar' is not bound here

local foo = function bar(n)
	-- 'bar' is bound here
end

-- 'bar' is not bound here
```

## Drawbacks

This proposal would make table function field syntax impossible without a special case. An example of this syntax:

```luau
local t = {
	function foo() end,
}

t.foo()
```

## Alternatives

The alternative is defining a function with `local function` declaration syntax, which allows for recursion and gives debugging names.
