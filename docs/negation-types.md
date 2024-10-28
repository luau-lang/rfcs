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

Choosing `unknown` as the basis makes a lot of things fall in place properly, including the property we wish to maintain where `~~T` is equivalent to `T`. It's crucial that we handle negation of error types properly, otherwise double negations won't actually be consistent. It'd allow error suppression to be improperly suppressed! See below, where `~~any` ends up as `unknown` instead of `any`.

1. `~~any`
2. `~~(unknown | *error*)`
3. `~(~unknown & ~*error*)`
4. `~(never & unknown)`
5. `~never`
6. `unknown` (not equivalent to `any`!)

As you can see, since through the series of rewrites `~~any` did not produce a type equivalent to `any`, our basis set must be `unknown`, and negation of an error suppression is still an error suppressing type. Therefore `~*error*` is `*error*`. This fixes the double-negation inconsistency:

1. `~~any`
2. `~~(unknown | *error*)`
3. `~(~unknown & ~*error*)`
4. `~(never & *error*)`
5. `~never | ~*error*`
6. `unknown | *error*` (`any` is an alias to `unknown | *error*`, so equivalence is achieved)

### Restrictions

We currently only support negation of _testable_ types. Intuitively, a testable type is one that can be tested by type refinements at runtime. This includes singleton types, primitive types, and classes. Of course, unions and intersections can be negated as well if and only if all of its constituent types are also testable types, due to the distributivity property of unions and intersections.

Since types like `{ p: P }` and `(T) -> U` are structural types and the runtime offers no way to test for them, they cannot be negated. An attempt to negate this will just turn the negation type into an error suppressed type. This means `~{ p: P }` and `~(T) -> U` are nonsensical. In the event that you negate some non-testable type, you get a type error.

Another restriction is that negation types cannot negate generic types. So `<T>(T) -> ~T` is also not legible.

### Implementation

Most of the implementation of negation types are already in place. The three things missing are:

1. the syntax,
2. some guarantee that no negation types survive if it negates non-testables, and
3. a type error if it negates a non-testable type in any way, shape, or form.

The parser change for this is trivial, so this is of no concern.

We can use type families to have this guarantee. Add a `negate<T>` type family which would be internal to the type inference engine, and have the syntax `~T` produce that type family, not a `NegationType`.

As for the type error, we can just consider `negate<{ p: P }>` to be an uninhabited type family, and resolve that as an error type.

## Drawbacks

Language design has a concept called weirdness budget. It's a fine line to balance, especially in this case where, even in popular type systems that have negation types, they are seldom used except by power users.

## Alternatives

We could provide a set of type aliases for some common use cases, e.g. `not_nil` for `~nil` and `not_string` for `~string` and so on. But this is limited, i.e. no way to negate for a specific string singleton except by a `typeof` hack.

Alternatively, provide a type family `not<T>`, which can be generalized to any type, so long as they obey the restrictions above! This alternative proposal is essentially the above proposal, sans syntax.

We could also use the set exclusion syntax `T \ U` which does provide an advantage of an explicit set to exclude a subset of, but there are downsides with this, e.g. it is neither commutative nor associative, which makes for the implementation more annoying since you cannot fold over them as easily.
