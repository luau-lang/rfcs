# Expose Built-in Type Functions to User-Defined Type Functions

## Summary

Exposes the built-in type functions present in luau such as `keyof` to the user-defined type function environment through the namespace `types.builtins`.

## Motivation

Currently, built-in type functions are not immediately accessible within the user-defined type function environment. This is not desired, as the built-in type functions provide utility such as `keyof` and `index`; and ways to handle operations between types with the likes of `add` and `unm`.

Today, the way to circumvent this is to use boilerplate aliases such as `type proxy_keyof<T> = keyof<T>` to be able to use the built-in type functions; which may seem strange for why these aren't readily available.

## Design

A new namespace called `builtins` under the `types` library is to be introduced. This namespace will contain all public-facing built-in type functions that are available.

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
-- type proxy_unm<T> = unm<T>
```

## Drawbacks

* There is no precedent to add sub-namespaces to existing libraries in the luau standard library, such as `types.builtins` as this RFC proposes.

* There is certain built-in type functions that might seem strange to see in the `types` library. One example would be the `setmetatable` built-in, since `tabletype:setmetatable` and `types.newtable`s `metatable` argument supersede this built-in. Users might get confused or distracted trying to discern if it serves a different purpose behaviourally/semantically.

* The name `builtins` does not indicate that it is a namespace that contains built-in *type functions* specifically.

## Alternatives

* Let users write aliases for the built-in type functions the way it currently can be done.

* Only expose certain built-in type functions *selectively*, instead of exposing all public-facing ones. This would allow superseded built-in type functions like `setmetatable` to be omitted entirely.
	* This alternative could also lead to other built-ins like `lt` to be renamed to `lessthan`, thus making `types.*` be a more viable alternative to `types.builtins.*`.

### Namespace Alternatives

* Dump all built-in type functions into the global namespace
	* This would be highly undesirable, since now the global namespace is polluted with both value and type functions. And certain labels would clash with already existing labels like `rawget`.

* `types.*`, '*' being the built-in name
	* This seems like the idiomatic alternative; although, names like `lt` and `unm` might feel off next to verbose (but concise) names like `intersectionof` and `newfunction`.

* `types.typefunctions.*`
	* The name `types.builtins` might not indicate that it contains built-in type functions specifically, this alternative could alleviate this issue. Although some may consider it too-verbose.

* `typefunctions.*`
	* This would fix the problem that comes with implementing `types.builtins` as there is no precedent in luau for such a thing. But the name `typefunctions` is rather ambigous whether it contains *all* type functions (UDTFs included) or rather just the built-in ones.

* `builtintypefunctions.*`
	* This fixes the ambiguity with `typefunctions` but is too verbose, due to this the readability and ergonomics might be impacted.
