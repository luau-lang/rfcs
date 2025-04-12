# Metatable Type Syntax

## Summary

Add unique syntax to allow for specifying the metatable of a type.

## Motivation

There is currently no expression for metatables within the type system, with the exception of hacks using `typeof`. That isn't great, because metatables are a commonly required feature for things like object-oriented programming. `typeof` is boilerplate which can likely be eliminated by allowing the expression of metatables through special syntax.

## Design

Introduce a context-sensitive `metatable` keyword which specifies that a table has a metatable.

```lua
local class = {}
type Identity = {
	field: string,

	metatable typeof(class)
}
```

This could cause confusion with table properties named `metatable`. To combat this, a lint could be added which will warn upon setting table properties named `metatable`. To silence this lint, `["metatable"]` would need to be used instead.

## Alternatives

`type T = { metatable {} }` could be too similar to field syntax. We could change the keyword to be `@metatable`, similar to how it is currently expressed as when converting a type to a string. However, this introduces a sigil, which increases cognitive load and harms readability.

We could instead rely on [metatable type functions](https://github.com/luau-lang/rfcs/pull/69) to achieve what this RFC proposes. However, relying purely on `setmetatable` for metatable type expression isn't ideal because it involves more verbose syntax than what this RFC proposes.

The final alternative is also to just do nothing, and live with the boilerplate code required by the `typeof` solution. This doesn't add any new functionality.

## Drawbacks

Unique syntax always has the downside of an additional learning curve to the language. Context-sensitive keywords and sigils increase cognitive load when trying to read Luau, which isn't great.
