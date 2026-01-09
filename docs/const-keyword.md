# Const Keyword

## Summary

Introduce a `const` keyword for local variable bindings that prevents reassignment after initialization. A const declaration makes the binding immutable, not the value, so tables and other mutable objects remain mutable through the binding.

## Motivation

When writing code, many variables are declared once and never intended to change such as module imports, function declarations, service references, and configuration objects. However, there is no way to prevent them from being rebound, which can lead to accidental rebinding, more time spent validating variables do not change during code reviews, and reducing the effectiveness of certain compiler / typechecker optimizations

Additionally, with the proposal of the [export keyword](https://github.com/luau-lang/rfcs/pull/42), variables declared with it are frozen in the module exports table and therefore expected to refer to stable values. Without const-ness they can accidentally be redefined in confusing ways, within the module definition.

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

`const` becomes a keyword in the `local` declaration position. If `const` is currently allowed as an identifier, this is a source-breaking change only for code that uses const as a local variable name in positions where a local declaration is parsed. (See Drawbacks.)

## Drawbacks

- Keyword and compatibility surface: if `const` is currently used as an identifier in existing code, reserving it as a keyword can be source-breaking in some contexts.
- Potential false sense of immutability: developers may misread const as implying deep immutability. This is mitigated by clear documentation and examples, but the risk remains.

## Alternatives

- Lint-only "never reassigned" rule: A linter can infer locals that are never reassigned and warn on future changes. This avoids new syntax, but loses the ability to declare intent directly in code (e.g., a new assignment becomes a warning rather than a clear violation of an explicit constraint).
- Deep immutability / freezing: Enforcing immutability of tables (recursively) would address mutation bugs but introduces complexity, runtime overhead, questions around metatables/userdata, and awkward interactions with existing patterns. This is explicitly out of scope for this proposal.
- Annotations instead of a keyword: An attribute-like syntax (comment directive or type-level modifier) could avoid reserving a keyword, but is less readable, less discoverable, and harder to standardize across tooling.
- Do nothing: Continue relying on convention and review. This keeps the language smaller but preserves a common footgun and prevents tooling from enforcing a widely useful invariant.
