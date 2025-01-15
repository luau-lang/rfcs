# Relax the recursive type restriction

## Summary

Relax the [recursive type restriction](recursive-type-restriction.md) by implementing isorecursive
types properly for parameterized type aliases.

## Motivation

Luau supports parameterized type aliases as follows:
```luau
type Box<T> = { data: T }
```

These type aliases can also be recursive:
```luau
type Tree<T> = { data: T, children: {Tree<T>} }
```
but the [recursive type restriction](recursive-type-restriction.md) limits us to supporting only
direct recursion as above, where the type aliases' parameters remain unchanged.

At the time the restriction was implemented, Luau featured a greedy type inference engine that would
eagerly expand every occurrence of a type alias, and, as such, would infinitely expand until
ultimately crashing from overflowing the stack, or would hit a recursion limiter and produce a
strange, very deep type.

This limitation means that Luau's type system is unable to express common recursive type patterns
for data structures, such as `andThen` for a monadic data type like `Promise`. For instance, the
following definition is rejected because of the use of `Promise<U>` in the signature of `andThen`:
```luau
type Promise<T> = {
    create: (T) -> Promise<T>,
    andThen: <U>(self: Promise<T>, callback: (T) -> Promise<U>) -> Promise<U>
}
```

## Design

Today, Luau has a new type inference engine that is able to deal with dynamic constraint resolution
and has already allowed us to build a system for arbitrary type functions, both builtin and
provided with the language and [user-defined type functions](user-defined-type-functions.md). We can
relax the recursive type restriction today in this new type inference engine by treating type
aliases as proper type functions that are lazily expanded when their contents are needed.

## Drawbacks

Eliminating this restriction means committing to an intrinsically more complicated architecture for
type inference in Luau in general. The Luau team, however, has already made the decision to do this,
and so there seems to be little drawback beyond the obvious implementation work required in making
type alias expansion completely lazy. This pays for itself in the considerable gain in expressivity
gained for users of the type system.

## Alternatives

The only serious alternative besides doing nothing at all would be to implement equirecursive types.
Equirecursive types have considerably greater complexity, both for humans and for computers. They
pose considerable algorithmic challenges for both type checking and type inference, and require
an implementation of a canonicalization of recursive types that is O(n log n). Isorecursive types
are simpler, map closely to what is already implemented in Luau today, and match the intuitions
users likely have from interacting with recursive types in nominal object-oriented languages.
