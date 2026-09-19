# Builtin Truthy & Falsey Type Aliases

## Summary

Introduce two built-in type aliases for `truthy` and `falsey` representing the truthiness in boolean-like or ternary operators (`if`, `and`, `or`, `not`, etc.)

## Motivation

Truthiness and falsiness is important for understanding control flow in conditional operations. Currently, the types representing truthy/falsey values are fairly obtuse (`~(false?)`, `false?`, or more verbosely `~(false | nil)` / `false | nil`). We can make these types easier for users to reason about, as well as easier to read and write.

## Design

This RFC proposes adding two built-in type aliases as definitions:

```luau
type truthy = ~(false | nil)
type falsey = false | nil
```

## Drawbacks

These new aliases may conflict with existing type names in user code, and they may also introduce some confusion for users who are not familiar with the concept of truthiness or falsiness.

## Alternatives

- do nothing. this would keep the language's built-in type aliases more minimal (only what's necessary), which makes the experience easier for new users. I personally believe the convenience and relatively common knowledge about truthiness and falsiness of values will outweigh the simplicity of not doing this.
- `falsy` instead of `falsey`. There's nothing wrong with this, but it has a different column width from `truthy`
- introduce `types.falsey` and `types.truthy` in the type function runtime. This can be discussed separately, since it is not exclusive with this RFC.
