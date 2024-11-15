# Shorter vector constructor

## Summary

Replace the current vector constructor `vector.create` with a less verbose, more ergonomic `vector`.

## Motivation

While the current vector constructor `vector.create` is consistent with other functions in the vector library, it is quite verbose. A shorter version would improve the readability and writability of code.

Especially scripts that contain a lot of data such as positions, directions, matrices or colors could be significantly less verbose with a shorter syntax.

## Design

The `__call` metamethod in the vector library table will be set to the already existing vector constructor. Thus calls to `vector` will create a new vector value. Any existing code that uses `vector.create` can swap in `vector` with identical behavior.

The new constructor `vector` shall be backed with a built-in fastcall so that `vector` and `vector.create` will have identical performance characteristics. As a result vectors will also be added to the constant table by the compiler when the new shorter version of `vector` is used.

The old constructor `vector.create` will be deprecated and eventually removed in favor of the new shorter version. The removal of `vector.create` simplifies the internal implementation as fastpaths for two constructors do not need to be supported.

## Drawbacks

The biggest drawback is the deprecation and removal of `vector.create` which requires end users to modify their code. During the transition time two different constructors would exist which could be slightly confusing.

The performance of constructing vectors could be negatively affected in some cases where the built-in fastcall is not available or when the fastcall is aborted. For example, when the constructor arguments are strings and the string to number auto-coersion path is taken. It is assumed that the performance in these corner cases is not important.

## Alternatives

Alternatively a new syntax for creating vectors could be added. For example, `|x y z|` or `<x y z>`. While this would reduce the verbosity even further, it would be a bigger change. One of the original design goals of Lua is simplicity and favoring words over symbols, so `vector(x, y, z)` is more aligned with that.

Alternatively we could keep the old constructor and add the new shorter version as an alias. To the end user it could be confusing to have two different functions for doing the same thing. This would also not be aligned with the Lua philosophy of keeping things simple.

If this feature is not implemented, we would not get the benefits discussed above. Additionally some embedders with an existing shorter constructor could shy away from the built-in library, which would work against the goals of having a unified, official vector library.