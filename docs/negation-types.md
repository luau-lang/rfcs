# Negation types

## Summary

A type that represents the complement of the type being negated, which is a union type of all inhabited types (`unknown`) excluding the type being negated.

## Motivation

Type refinements will produce negation types to get the complementation. For the most part, users will never need to write these negation types themselves, except for when they want to _maintain_ some invariants beyond the scope of the branch.

A [cofinite set](https://en.wikipedia.org/wiki/Cofiniteness) arises when you have some negated `X` in conjunction with some supertype of `X`. For example, `string & ~"a"` is a cofinite string set that accepts any `string` excluding `"a"`. This happens often with type refinements where `typeof(x) == "string" and x ~= "a"` gives rise to the type `string & ~"a"`.

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

### Implementation

The parser change for this is trivial, so this is of no concern.

Most of the real work would be spent on revising the type system to handle negation types of non-testable types in normalization, simplification, and subtyping rules, as well as adjusting `types.negationof` to allow function types and table types.

## Drawbacks

Language design has a concept called weirdness budget. It's a fine line to balance, especially in this case where, even in popular type systems that have negation types, they are seldom used except by power users.

## Alternatives

We could provide a set of type aliases for some common use cases, e.g. `not_nil` for `~nil` and `not_string` for `~string` and so on. But this is limited, i.e. no way to negate for a specific string singleton except by a `typeof` hack.

Alternatively, provide a type family `not<T>`, which can be generalized to any type! This alternative proposal is essentially the same as the proposal, sans syntax. Unless we rename it to `negate`, this still requires changing the parser anyway since `not` is a keyword.

We could also use the set exclusion syntax `T \ U` which does provide an advantage of an explicit set to exclude a subset of, but there are downsides with this, e.g. it is neither commutative nor associative, which makes for the implementation more annoying since you cannot fold over them as easily.
