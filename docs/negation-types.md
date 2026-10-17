# Negation types

## Summary

A type that represents the complement of the type being negated, which is a union type of all inhabited types (`unknown`) excluding the type being negated.

## Motivation

Type refinements will produce negation types to get the complementation. For the most part, users will never need to write these negation types themselves, except for when they want to _maintain_ some invariants beyond the scope of the branch, or if the user has overloaded a specific type in a forall quantifier. For example, React has a function called `useState` whose signature is `<S>(state: (() -> S) | S, ...) -> (S, Dispatcher<BasicStateAction<S>>)`. The problem here is that both `S` and `() -> S` are valid unifiers for the function `() -> T`, so you end up with this monstrosity:

```
(state: (() -> (() -> T) | T) | (() -> T) | T) -> ((() -> T) | T, Dispatcher<BasicStateAction<(() -> T) | T>>)
```

Another consequence of this signature: functions whose arity does not satisfy the expected arity of `() -> S` will go into `S` which defeats the whole point. This is why when you write `if typeof(state) == "function" then state() else state`, you get a type error at `state()`: ``Cannot call a value of type `S & function` in union: `(() -> S) | (S & function)` ``. The current workaround is `(state :: () -> S)()` which is unsound. The counterexample:

```luau
local function useState<S>(state: (() -> S) | S): S
  return if typeof(state) == "function"
    then (state :: () -> S)()
    else state
end

local function foo(x: number)
  return x + 1
end

-- s: ((x: number) -> number) | number
local s = useState(foo) -- no type errors here
```

This program passes the type checker like nothing is wrong, but it's clear that `S` in this case is intended to be the _residual_ of whatever the unifier of `() -> S` is, and a residual is a complement, so the correct signature for `useState` requires negation types: `<S: ~(...unknown) -> ...unknown>(state: (() -> S) | S, ...) -> (S, Dispatcher<BasicStateAction<S>>)`.

...which is perfectly valid use case of negation types. By allowing negation types on non-testable types, the intersection `S & function` becomes absurd from the bound `S: ~(...unknown) -> ...unknown`, so `S & function` normalizes to `never`, leaving `state` with the type `() -> S` in the then branch, and the type `S` in the else branch. We can also now reject code that has no unifiers for `() -> S` and `S: ~(...unknown) -> ...unknown` like `useState(foo)`, and type inference behaves as expected for these overloaded polymorphic functions where you are expected to pass in either a function of the signature `() -> T` _or_ a value of type `T` and nothing in-between.

## Design

### Syntax

The notation is `~T` where `T` is a type annotation. It binds tightly to the type it negates, so `~T | U` is `(~T) | U`, not `~(T | U)`. There's really not much to it, syntax-wise. It has the same precedence as the `not` expression, which is why `~T | U` is `(~T) | U` in the same way `not t or u` is `(not t) or u`.

We propose to allow users to write such a type. Formally:

```ebnf
ty ::= `~` ty
     | name [`<` [ty {`,` ty}] `>`]
     | ty {`|` ty}
     | ty {`&` ty}
     | `(` ty `)`
     | `typeof` `(` exp `)`
     | table_ty    (* omitted for brevity *)
     | function_ty (* omitted for brevity *)
```

### Semantics

The basis set we want to perform set exclusion on is `unknown`, _not_ `any`. This means given the type `~"a"`, it is equivalent to the type `unknown & ~"a"`.

A [cofinite set](https://en.wikipedia.org/wiki/Cofiniteness) arises when you have some negated `X` in conjunction with some supertype of `X`. For example, `string & ~"a"` is a cofinite string set that accepts any `string` excluding `"a"`. This happens often with type refinements where `typeof(x) == "string" and x ~= "a"` gives rise to the type `string & ~"a"`.

### Implementation

The parser change for this is trivial, so this is of no concern.

Most of the real work would be spent on revising the type system to handle negation types of non-testable types in normalization, simplification, and subtyping rules, as well as adjusting `types.negationof` to allow function types and table types.

## Drawbacks

Language design has a concept called weirdness budget. It's a fine line to balance, especially in this case where, even in popular type systems that have negation types, they are seldom used except by power users.

## Alternatives

We could provide a set of type aliases for some common use cases, e.g. `not_nil` for `~nil` and `not_string` for `~string` and so on. But this is limited, i.e. no way to negate for a specific string singleton except by a `typeof` hack.

Alternatively, provide a type family `not<T>`, which can be generalized to any type! This alternative proposal is essentially the same as the proposal, sans syntax. Unless we rename it to `negate`, this still requires changing the parser anyway since `not` is a keyword.

We could also use the set exclusion syntax `T \ U` which does provide an advantage of an explicit set to exclude a subset of, but there are downsides with this, e.g. it is neither commutative nor associative, which makes for the implementation more annoying since you cannot fold over them as easily.
