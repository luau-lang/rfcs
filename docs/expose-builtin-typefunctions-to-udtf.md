# Expose Built-in Type Functions to User-Defined Type Functions

## Summary

Exposes the built-in type functions present in Luau such as `keyof` to the user-defined type function environment through a `builtins` submodule of the `types` library.

## Motivation

Currently, built-in type functions are not immediately accessible within the user-defined type function environment. We would like to be able to use them because the built-in type functions include both utilities like `keyof` and `index`,  and functions to resolve the types of overloadable operators like `add` and `unm`.

Since user-defined type aliases _are_ exposed to type functions, you can workaround the existing limitation by defining a new alias like `type aliasKeyOf<T> = keyof<T>`. However, we find this workaround to be unnecessarily cumbersome since it must be repeated in each module the user might define a type function using that builtin.

## Design

We propose introducing a new submodule called `builtins` under the `types` library. This submodule will contain all public-facing built-in type functions that are currently available.

An example:
```luau
type function ultra_negate(t: type): type
	return types.negationof(
		types.builtins.unm(t) --applies the unary minus (-) operator to the type
	)
end

type negatable_string = setmetatable<{}, {
	__unm: (negatable_string) -> (string)
}>

type foo = ultra_negate<negatable_string> -- 'foo' resolves to '~string'

-- * this example previously would have needed to use an alias for 'unm' like seen below
-- type aliasUnm<T> = unm<T>
```

## Drawbacks

* There is no existing precedent of adding submodules to libraries in the Luau standard library. This RFC proposes to do exactly that with `types.builtins`.

* Some built-in type functions might be a source of confusion due to overlap with functionality in the `types` library itself. One example would be `setmetatable` which ultimately shares the same behavior as `tabletype:setmetatable`. It may not be immediately obvious whether or not their behaviors differ at all, or when someone should use one or the other.

* The name `builtins` does not indicate that it is a submodule containing built-in _type functions_ specifically.

## Alternatives

* Do nothing. This means any uses of built-in type functions will continue to be through aliases introduced by the use.

* Only expose certain built-in type functions _selectively_, instead of exposing all public-facing ones. This would allow potentially confusing or redundant builtin type functions like `setmetatable` to be omitted entirely.

### Naming Alternatives

* Dump all built-in type functions into the global scope
	* This would be highly undesirable, since now the global namespace is polluted with both value and type functions. And certain labels would clash with already existing labels like `rawget`.

* `types.*`, '*' being the built-in name
	* This seems like the idiomatic alternative; although, names like `lt` and `unm` might feel off next to verbose (but concise) names like `intersectionof` and `newfunction`.
	* Mixing built-in type functions directly into the `types` library might be undesirable, because the `types` library specifically contains type functions with _known_ return types. For example, when `types.optional` is used, the return type is guaranteed to be an `optionaltype`. In contrast, built-in type functions (especially those that resolve overloadable operators) can evaluate to _any_ arbitrary `type`.

* `types.typefunctions.*`
	* The name `types.builtins` might not indicate that it contains built-in type functions specifically, this alternative could alleviate this issue. Although some may consider it too-verbose.

* `typefunctions.*`
	* This would fix the problem that comes with implementing `types.builtins` as there is no existing precedent for nested submodules. However, the name `typefunctions` is ambiguous as to whether it contains _all_ type functions (UDTFs included), or just the built-in ones.

* `builtintypefunctions.*`
	* This fixes the ambiguity with `typefunctions` but is too verbose, due to this the readability and ergonomics might be impacted.

<!--
This was a previous alternative proposed, though has been removed due to it's redundancy (i think), here's why:
1. the built-in type functions are already accessible with names like `lt`. this naming scheme is already consistent with metamethods.
2. this inconsistency in the name would also be confusing due to, well, it's inconsistency in the name... leading to questions about whether these two things (e.g. `lt` and `lessthan`) are actually 1:1 or not!
3. unnecessary burden for language developers to define new names for these functions, not the biggest deal ever, but considering the previous two points, this alternative would most likely seem redundant more than anything.

* Only expose certain built-in type functions selectively _and_ rename certain built-in type functions like `lt` to be more verbose like `lessthan`. The main appeal for this would be that, if the naming alternative `types.*` was to be preffered, short and potentially cryptic names like `lt` could look off next to more verbose names like `intersectionof` as was mentioned in the `types.*` alternative; renaming would thus make `types.*` be prefferable.
-->
