# Const Keyword

## Summary

Introduce a `const` keyword for local variable bindings that prevents reassignment after initialization. A const declaration makes the binding immutable, not the value, so tables and other mutable objects remain mutable through the binding.

## Motivation

When introducing the [`export`](https://github.com/luau-lang/rfcs/pull/42) keyword we will need to introduce the concept of `const`-ness to the language. Modules with exports return a frozen table of values; however, the variables used to declare those exports are not themselves immutable. This makes it possible to write code such as the following:

```luau
export foo = 5

export function increment()
	foo += 1
end
```

This results in different values for `foo` depending on where it is accessed from:

```luau
-- Inside the module
print(foo) --> 5
increment()
print(foo) --> 6

-- Outside the module
print(module.foo) --> 5
module.increment()
print(module.foo) --> 5
```

To address this, we propose that all exported values are `const` by default, preventing reassignment within the module and ensuring that exported bindings represent stable values.

However, this is not the only use-case for `const`. A `const` binding provides a clear guarantee that a name will always refer to the same value for the lifetime of its scope. This property is useful beyond module exports: it makes code easier to reason about, avoids accidental rebinding in closures or long-lived scopes, and allows tools and the typechecker to treat such bindings as stable references rather than values that may change over time. Since `export` already requires this guarantee to behave correctly, exposing `const` as a general language feature avoids introducing special-case rules and provides a simple, consistent way to express immutability of bindings where it is desired.

## Design
### Syntax

Allow `const` in declarations anywhere `local` is allowed:

```luau
const x = 5
const f = function() end
const t = { a = 1 }
```

Typed declarations remain valid:

```luau
const x: number = 5
const t: { a: number } = { a = 1 }
```
Multi-assignment is also supported:

```luau
const a, b = 1, 2
```

A `function` can be declared as `const`:

```luau
const function f()
  -- do something
end
```

### Semantics

A `const` variable is equivalent to a variable declared with `local` however `const` variables cannot be reassigned after they are initialized:

```luau
const x = 1
x = 2 -- error
```

This includes any assignment form that writes to the binding, such as compound assignment:

```luau
const x = 1
x += 1 -- error
```

It applies to the variable's binding, not to the value it refers to. If a const variable refers to a mutable value such as a table, that value can still be modified:

```luau
const t = { count = 0 }
t.count += 1 -- ok (mutating the table)
t = {}       -- error (reassigning the binding)
```

`const` is not a replacement for `table.freeze` which should be used to prohibit mutation of table contents. The two features address different problems and can be used together:

```luau
const t = table.freeze({
	count = 0,
})

t.count += 1 -- error (mutating a frozen table)
t = {}       -- error (reassigning the binding)
```

#### Initialization

A `const` binding must be initialized at the point of declaration:

```
const x -- error: const variable must be initialized
```

This keeps the feature simple, avoids "definite assignment" rules, and makes the "never reassigned" intent explicit.

#### Scope and Shadowing

`const` follows normal lexical scoping rules. Shadowing creates a new binding and is allowed:

```luau
const x = 1
do
	const x = 2 -- ok: new binding in inner scope
end

const y = 1
const y = 2 -- also ok: rebinding as const in same scope
```

#### Captures / Up-Values

If a `const` variable is captured by an inner function, it remains non-reassignable from within that function as well:

```luau
const x = 1
const function f()
	x = 2 -- error
end
```

### Grammar / Parsing

`const` is a contextual keyword that is only valid in positions where `local` is valid. This makes the introduction fully backwards compatible with existing code.

## Drawbacks

- Potential false sense of immutability: developers may misread const as implying deep immutability. This is mitigated by clear documentation and examples, but the risk remains.
- Lua introduced `const` in the form `local foo <const> = 5` which is inconsistent with this syntax. However, we've already decided we won't adopt Lua const syntax, therefore consider this an acceptable change.

## Alternatives

- Lint-only "never reassigned" rule: A linter can infer locals that are never reassigned and warn on future changes. This avoids new syntax, but loses the ability to declare intent directly in code (e.g., a new assignment becomes a warning rather than a clear violation of an explicit constraint).
- Deep immutability / freezing: Enforcing immutability of tables (recursively) would address mutation bugs but introduces complexity, runtime overhead, questions around metatables/userdata, and awkward interactions with existing patterns. This is explicitly out of scope for this proposal.
- Annotations instead of a keyword: An attribute-like syntax (comment directive or type-level modifier) could avoid reserving a keyword, but is less readable, less discoverable, and harder to standardize across tooling.
- Do nothing: Continue relying on convention and review. This keeps the language smaller but preserves a common footgun and prevents tooling from enforcing a widely useful invariant.
