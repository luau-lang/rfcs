# Match Expression

## Summary

This RFC proposes a `match` expression for Luau.

The goal is to make multi-branch value selection clearer when logic depends on one or more values and branch order matters.

The construct supports:

- matching against one or more values
- positional patterns for multi-value matching
- wildcard `_` patterns
- expression-list results (including multiple return values)

This is intended as a structured alternative to repeated `if` / `elseif` comparisons in expression position.

## Motivation

Luau already supports `if` expressions:

```lua
local result = if a == 1 then
	5
elseif b == 2 then
	6
else
	7
```

For single conditions this is fine. For multi-value branching, conditions become repetitive and less readable:

```lua
local result
if a == 1 and b == 0 then
	result = 5
elseif a == 0 and b == 2 then
	result = 6
else
	result = 7
end
```

The compared values (`a`, `b`) repeat, and branch priority is spread across boolean logic.

A `match` expression centralizes compared values once and expresses branches positionally.

A concrete case is leap-year logic:

```lua
local mod400 = year % 400
local mod100 = year % 100
local mod4 = year % 4

local isLeapYear = match mod400, mod100, mod4 with
	if 0, _, _ then true
	elseif _, 0, _ then false
	elseif _, _, 0 then true
	else false
end
```

This directly encodes the precedence rules:

- divisible by 400 -> leap year
- divisible by 100 -> not leap year
- divisible by 4 -> leap year
- otherwise -> not leap year

## Design

### Syntax

The syntax takes its inspirations from OCaml.

Canonical form:

```lua
match <expr-list> with
	if <pattern-list> then <expr-list>
	elseif <pattern-list> then <expr-list>
	else <expr-list>
end
```

Rules:

- `<expr-list>` is one or more expressions separated by `,`
- `<pattern-list>` is one or more patterns separated by `,`
- each branch must provide exactly one pattern per matched expression
- in this revision, patterns are limited to literals and `_`

### Examples

Single value:

```lua
local speed = match state with
	if "Idle" then 0
	elseif "Run" then 16
	else 8
end
```

Multiple values:

```lua
local value = match a, b with
	if 1, 0 then 5
	elseif 0, 2 then 6
	else 7
end
```

Wildcard:

```lua
local value = match a, b with
	if 1, _ then 5
	elseif _, 2 then 6
	else 7
end
```

Multiple return values:

```lua
local x, y = match a, b with
	if 5, _ then "A", "B"
	elseif _, 5 then "B", "A"
	else "C", "C"
end
```

Use in return position:

```lua
local function f(x)
	return match x with
		if "a" then "b"
		else "c"
	end
end
```

### Semantics

- Match input expressions are evaluated once, left-to-right.
- Cases are tested top-to-bottom.
- A literal pattern matches by equality (`==`).
- `_` matches any value and does not bind.
- The first matching case is selected.
- If no case matches, the `else` case is used when present.
- If no case matches and `else` is omitted, this is a runtime error.
- The selected case yields an expression list, which is the value of the `match` expression.

For multi-value matching:

```lua
match a, b with
	if p1, p2 then ...
```

is equivalent to checking:

```lua
a == p1 and b == p2
```

with `_` acting as an always-true position.

### Type Checking

`match` integrates with existing branch-based analysis.

Literal narrowing (single value):

```lua
type State = "Idle" | "Walk" | "Run"

local speed = match state with
	if "Idle" then 0
	elseif "Walk" then 8
	elseif "Run" then 16
	else 0
end
```

Within each branch, matched values may be narrowed to literal-compatible types.

Multi-value narrowing:

```lua
type A = 1 | 2
type B = 3 | 4

local result = match a, b with
	if 1, 3 then "x"
	elseif 2, 4 then "y"
	else "z"
end
```

Each position can narrow independently.

Exhaustiveness:

- When patterns are literals over finite types, implementations may report non-exhaustive matches if `else` is omitted.
- `else` suppresses exhaustiveness diagnostics.
- `_` in all remaining uncovered positions behaves as a catch-all.

Wildcard behavior:

- `_` does not introduce bindings and does not itself narrow matched values.

Result type:

- The result type is the union of branch result types.
- For multi-value returns, branches should have consistent arity when statically known.
- If arity differs, implementation may report an error or apply existing Luau multi-return behavior.

### Desugaring (Conceptual)

```lua
match a, b with
	if 1, _ then x
	elseif _, 2 then y
	else z
end
```

is conceptually equivalent to:

```lua
local __m1, __m2 = a, b

if __m1 == 1 then
	x
elseif __m2 == 2 then
	y
else
	z
end
```

This is an explanatory model, not a required lowering strategy.

## Restrictions

- `match` is an expression and must be used in expression position.
- Raw varargs (`...`) are not supported as match input.
- Patterns in this revision are limited to literals and `_`.

## Rationale

Why add syntax instead of only using `if` expressions?

- It removes repeated comparison targets in multi-value branching.
- It makes positional structure explicit.
- It preserves explicit first-match ordering.

Why reuse `if` / `elseif` / `else` tokens?

- The branch model is familiar.
- It avoids introducing a second unrelated case-branch keyword family.
- It keeps parser and reader expectations close to existing Luau control flow.

Why allow expression-list results?

- Luau already supports multiple return values.
- `match` composes naturally with that model in assignments and returns.

## Alternatives

### `if` expressions

Equivalent logic is possible with chained conditions, but requires repeated inputs and does not express multi-value matching positionally.

### Table-based dispatch

For simple one-key lookups, tables are excellent:

```lua
local speedByState = {
	Idle = 0,
	Walk = 8,
	Run = 16,
}

local speed = speedByState[state] or 0
```

For ordered multi-value logic with wildcards and computed multi-value results, table dispatch becomes indirect and often needs nested structures plus helper code.

### `switch` construct

A single-value `switch` can improve one-input dispatch, but does not directly model positional multi-value cases.

Representing this with nested switches increases indentation and obscures first-match intent across positions.

`match` addresses this by expressing all compared values and branch patterns in one construct.

## Drawbacks

- Introduces new expression syntax and parser surface area.
- Partially overlaps with `if` expressions for simple cases.
- Requires editor and formatter support for a new block-like expression form.
- Initial pattern language is intentionally limited (literals and `_`).

## Future Work

Possible extensions:

- binding patterns
- table/shape patterns
- guarded patterns
- vararg matching
- stricter exhaustiveness diagnostics

## Conclusion

`match` provides a small, explicit construct for ordered multi-value branching with wildcard support and expression-list results.

It keeps semantics close to existing Luau behavior while improving readability and maintainability in cases where chained `if` logic or table dispatch becomes hard to follow.
