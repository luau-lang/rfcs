
# Comparison Chaining

## Summary

Allowing the `<`, `<=`, `>`, and `>=` operators to be chained successively.

## Motivation

Checks between bounds commonly repeat a value with two comparisons:
```luau
if min <= value and value <= max then
    -- etc
end
```

Comparison chaining expresses the same condition more clearly:
```luau
if min <= value <= max then
    -- etc
end
```

Repeating an expression can also repeat work or produce different values. For example, checking the result of a function against two bounds currently requires storing its result when it has to be evaluated only once:
```luau
local value = getValue()
if minimum <= value and value < maximum then
    -- ...
end
```

With chaining, the condition can be written directly:
```luau
if minimum <= getValue() < maximum then
    -- ...
end
```

## Design

A chain compares each pair of adjacent operand values from left to right. Each operand is evaluated only once, and evaluation short-circuits at the first false comparison. The result is `true` if every comparison succeeds, and `false` otherwise.

For example:
```luau
return first() < middle() <= last()
```

is equivalent to:
```luau
local a = first()
local b = middle()
return a < b and b <= last()
```

Mixed directions are allowed: `a < b > c` checks `a < b` followed by `b > c`, reusing the evaluated value of `b`.

Each comparison follows any existing type checking and metamethod rules for its designated operator.

Parentheses preserve explicit grouping. `(a < b) < c` remains a comparison between the boolean result of `a < b` and `c` (which is an error).

Only `<`, `<=`, `>`, and `>=` participate in chaining. The grouping of expressions with no parentheses mixing chains with `==` or `~=` remains to be decided.

## Drawbacks

Chaining introduces evaluation rules that readers and language tooling must understand. Long chains, particularly those with mixed directions, may be harder to read.

## Alternatives
- Restrict chains to a consistent direction. This would exclude expressions such as `a < b > c`, despite their meaning following the same pairwise rule.
- Include `==` and `~=` in chains. This would broaden the feature beyond ordering comparisons and introduce additional expectations about expressions such as `a ~= b ~= c`.

## Prior Art

[Python](https://docs.python.org/3/reference/expressions.html#comparisons) supports comparison chaining with short-circuiting, single evaluation of shared operands, and mixed directions. It also allows equality, identity, and membership operators in chains.
