# Type Pack RFC

## Summary

This RFC proposes extending Luau type packs with reusable type pack aliases, pack expressions, and a dedicated `pack` runtime instance for user-defined type functions.

A `pack` represents an ordered sequence of types, optionally with component names and a variadic tail. Type packs should be constructible, inspectable, transformable, and usable through ordinary type aliases:

```luau
type Params = (name: string, age: number)
```

## Motivation

Type packs are already part of Luau's type system, but they are mostly visible through function types, variadic arguments, return types, and generic packs. This can make them feel like an implementation detail rather than a reusable part of the type language.

Exposing type packs more directly would make the type system clearer and more consistent. A function parameter list, a return list, and a generic pack would all be representable as the same kind of ordered type-level value.

This also gives user-defined type functions a direct way to inspect and preserve pack structure, including component order, optional names, and variadic tails.

## Design

### Type pack aliases

This RFC does not introduce a separate `type pack` declaration.

Instead, ordinary type aliases may refer to type packs when the right-hand side is a pack expression:

```luau
type Params = (name: string, age: number)
```

This defines `Params` as a type pack containing two ordered components:

```luau
(name: string, age: number)
```

A type pack alias can be used in a type pack position:

```luau
type Params = (name: string, age: number)
type Callback = (Params) -> ()
```

This is equivalent to:

```luau
type Callback = (name: string, age: number) -> ()
```

When a type pack alias contains a single component, it can be used either as the entire parameter pack or inside a parenthesized parameter list:

```luau
type Named = (name: string)

type CallbackA = Named -> ()
type CallbackB = (Named) -> ()
```

Both function types are equivalent to:

```luau
type Callback = (name: string) -> ()
```

Component names are preserved as metadata, but they do not affect type compatibility:

```luau
type Named = (name: string, age: number)
type Unnamed = (string, number)
```

The two packs have the same component types. The names are useful for tooling, diagnostics, documentation, and type functions, but they are not part of assignability.

### Single-component aliases

A parenthesized single component keeps its existing meaning in ordinary type positions.

```luau
type Grouped = (string)
```

This is equivalent to:

```luau
type Grouped = string
```

A named single-component alias behaves the same way in ordinary type positions:

```luau
type Named = (value: string)
```

This is treated as the underlying type:

```luau
type Named = string
```

However, when the same alias is used in a type pack position, it expands as a single-component pack and preserves its component name:

```luau
type Named = (value: string)

type CallbackA = Named -> ()
type CallbackB = (Named) -> ()
```

Both are equivalent to:

```luau
type Callback = (value: string) -> ()
```

### Variadic packs

Type pack aliases may include a variadic tail:

```luau
type LogArgs = (level: string, ...string)
```

Generic type packs remain valid:

```luau
type Callback<T...> = (T...) -> ()
```

Type pack aliases can be passed through generic pack positions:

```luau
type Params = (name: string, age: number)
type Callback<T...> = (T...) -> ()

type UserCallback = Callback<Params>
```

### Pack expressions

A pack expression is a parenthesized ordered list of pack components:

```luau
(string, number)
```

Components may optionally have names:

```luau
(name: string, age: number)
```

A pack expression may include a tail:

```luau
(prefix: string, ...number)
```

Pack expressions are valid in type pack positions, such as type pack aliases, function parameters, and function returns:

```luau
type Params = (name: string, age: number)
type Callback = (Params) -> ()
type Returns = () -> (success: boolean, message: string)
```

### Pack runtime instance

This RFC introduces a new type-function runtime instance: `pack`.

A `pack` represents an ordered list of components and an optional tail.

```luau
type PackComponent = {
	name: string?,
	type: type,
}
```

A `pack` instance supports the following methods:

```luau
pack:is(kind: "pack"): boolean

pack:components(): { PackComponent }
pack:getcomponent(index: number): PackComponent?
pack:getcomponentbyname(name: string): PackComponent?

pack:setcomponent(index: number, data: PackComponent)
pack:setcomponentbyname(name: string, data: PackComponent)

pack:tail(): pack? | type?
pack:settail(tail: pack? | type?)
```

The `components()` method returns an ordered array, not a dictionary. This is important because function parameters and return values are ordered.

The `setcomponent` method replaces a component at an existing index:

```luau
pack:setcomponent(1, {
	name = "userId",
	type = types.string,
})
```

This RFC intentionally does not include insertion or removal methods. Type functions can transform component types and metadata, but changing the arity of a pack should be handled by constructing a new `pack` value rather than mutating an existing one in place.

### Constructing packs in type functions

Type functions should be able to construct new packs:

```luau
type function Optionalize<P...>(params: P...)
	local result = types.newpack()

	for index, component in params:components() do
		result:setcomponent(index, {
			name = component.name,
			type = types.optional(component.type),
		})
	end

	result:settail(params:tail())

	return result
end
```

### Function type APIs

Function type runtime instances should expose their parameter and return type packs as `pack` instances.

For example:

```luau
type function ParamsOf<F>(callback: F)
	if not callback:is("function") then
		return types.never
	end

	return callback:parameters()
end

type Callback = (name: string, age: number) -> boolean
type Params = ParamsOf<Callback>
```

`Params` is equivalent to:

```luau
(name: string, age: number)
```

## Backward compatibility

Existing type pack syntax remains valid:

```luau
type Callback<T...> = (T...) -> ()
```

Existing function types remain valid:

```luau
type Callback = (string, number) -> ()
```

Existing type aliases remain valid:

```luau
type Name = string
```

Grouped type syntax remains valid:

```luau
type Name = (string)
```

This RFC adds the ability for type aliases to represent pack expressions and for type functions to work with packs through a dedicated `pack` runtime instance.

## Drawbacks

This feature adds another kind of type-function runtime instance, increasing the surface area of the type-function API.

It also introduces a new interpretation for some type aliases. For example:

```luau
type Params = (string, number)
```

would now define a type pack alias rather than being rejected or interpreted as another construct.

Single-component aliases may also be confusing because:

```luau
type T = (string)
```

is equivalent to the underlying type in ordinary type positions:

```luau
type T = string
```

but can still expand as a single-component pack in type pack positions. This distinction needs to be clearly documented.

Another drawback is that named pack components may create expectations that names affect type compatibility. This RFC explicitly treats names as metadata only, but users may still misunderstand this behavior.

This RFC intentionally avoids mutation operations that change pack length, such as inserting or removing components. This keeps the initial API smaller, but it means type functions that need to add or remove parameters must construct a new pack instead of modifying an existing one in place.

Finally, implementing this RFC may require changes to type alias resolution, type-function runtime representation, function type APIs, generic pack handling, and error reporting.

## Alternatives

### Separate `type pack` declarations

One alternative is to introduce a dedicated declaration form:

```luau
type pack Params = (name: string, age: number)
```

This would make type pack aliases explicit, but it would also add a new declaration kind. This RFC avoids that by reusing ordinary type aliases.

## Open questions

Should duplicate component names be allowed in a type pack?

If duplicate names are allowed, `getcomponentbyname` needs a clear rule for which component it returns. If duplicate names are rejected, the type checker needs to report that consistently for both source syntax and type-function-created packs.

Should `setcomponentbyname` be allowed to rename the component?

For example:

```luau
pack:setcomponentbyname("age", {
	name = "years",
	type = types.number,
})
```

This RFC allows the replacement data to contain a different name, but this may be surprising. An alternative is to require the replacement name to match the lookup name.

Should mutating methods modify a pack in place or return a new pack?

The examples in this RFC use in-place mutation. A more persistent API could avoid accidental sharing, but may be less ergonomic for type functions.

Should `tail()` return `pack? | type?`, or should variadic tails use a separate runtime representation?

A simple variadic tail such as `...string` can be represented as a `type`, while a generic or structured tail may need a `pack`. This RFC leaves the exact representation open.
