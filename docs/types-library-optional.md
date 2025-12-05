# types.optional

**Status**: Implemented

## Summary

This RFC proposes adding a new method to the `types` library in type functions, that provides a shorthand for `types.unionof(type, types.singleton(nil))`.

## Motivation

Optional types occur very commonly, and it is annoying to have to always type `types.unionof(meow, types.singleton(nil))` to make a optional type, this leads to having the following appear in codebases that make use of optional types a lot in type functions:

```luau
type function optional(type)
	return types.unionof(type, types.singleton(nil))
end
```

## Design

As a solution, a new method to the `types` library should be added called `optional`. When called on a union, this will add `nil` to the components. Otherwise, this new method will act the same as `types.unionof(type, types.singleton(nil))`.
