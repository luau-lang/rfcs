# Export Keyword

## Summary

Extend the `export` keyword to support values and functions.

## Motivation

Today, it is possible to export a type from a module with the `export` keyword:

```luau
export type Point = { x: number, y: number }
```

However, this mechanism is currently limited to types. Extending `export` to values and functions would provide a consistent way to expose a stable API from a module, while avoiding the extra boilerplate of returning a table.

## Design

### Syntax

Allow `export` in declarations anywhere that defines a name (`local`s, `function`s, and `type`s), but only at the top-level of a module:

```luau
export version = "5.1"

export function init()
	-- do a thing
end

export type Point = { x: number, y: number }
```

`export` for values is not permitted inside function bodies or other non–top-level scopes. Doing so results in a parse error.

### Semantics

#### Const By-Default

Exported values and functions are implicitly [`const`](https://github.com/luau-lang/rfcs/pull/166). This means that exports must be initialized at the point of declaration and cannot be reassigned after initialization.

```luau
export foo = 5
foo = 6 -- error: exported bindings are const
```

Without this, it would be possible to mutate the variable without changing to the exported table, leading to the following foot-gun:

```luau
export foo = 5

export function increment()
	foo += 1 -- 'foo' won't change in the exported table
end
```

#### Export Table Construction

Rather than populating an export table incrementally, the set of exported bindings is collected during parsing. When module execution completes, the module implicitly returns a frozen table containing the exported bindings.

For example, the following module:

```luau
export type Point = { x: number, y: number }

export function distance(a: Point, b: Point)
	local x, y = a.X - b.X, a.Y - b.Y
	return math.sqrt(x * x + y * y)
end
```

Can be considered sugar for:

```luau
export type Point = { x: number, y: number }

const function distance(a: Point, b: Point)
	local x, y = a.X - b.X, a.Y - b.Y
	return math.sqrt(x * x + y * y)
end

return table.freeze({
	distance = distance,
})
```

#### Export Shorthand

For convenience, we provide a shorthand form of `export` which will export an existing binding with the same name:

```luau
const foo = computeFoo()
export foo
```

If `foo` was declared as `local` instead of `const` then subsequent uses of it here will be treated as if it were declared as `const`.

#### Order of Declarations and Mutual Dependencies

As with `local` and `const` exports are evaluated in source order. Using the shorthand from above, mutually recursive or dependent functions can be declared before they are exported.

```luau
function a()
	return b()
end

function b()
	return 42
end

export a
export b
```

Exporting before a binding is initialized is an error. Supporting hoisted exports or forward declarations is out of scope for this proposal and may be explored in a future RFC.

#### Calling Exported Functions Internally

Exported functions are still available within the module and can be called normally:

```luau
export function distance(a: Point, b: Point)
	return math.sqrt((a.X - b.X)^2 + (a.Y - b.Y)^2)
end

distance({0, 0}, {1, 1})
```

#### Nested Tables

Exporting a table exports the binding, not the contents of the table:

```luau
export triangle = {}

function triangle.draw()
	-- ...
end
```

The `triangle` binding is immutable, but the table itself is mutable unless explicitly frozen.

#### Returns

A module that uses `export` for values or functions may not also return a custom value. Attempting to do so is an error:

```luau
export function distance(a: Point, b: Point)
	return math.sqrt((a.X - b.X)^2)
end

return { distance = distance } -- error
```

Type-only exports may continue to coexist with an explicit return value, as they do today.

## Drawbacks

* Introduces another way to define module exports alongside explicit return tables.
* Requires exported bindings to be immutable, which may require restructuring some existing patterns.
* Extends the surface area of the `export` keyword beyond types.

## Alternatives

- Modules could implicitly export all top-level bindings. This is not backwards compatible, reduces explicitness, and complicates reasoning about module APIs.
- Modules can already export values by returning tables explicitly. This proposal is a quality-of-life and consistency improvement but is not strictly necessary.
