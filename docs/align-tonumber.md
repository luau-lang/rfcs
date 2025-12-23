# Align `tonumber` with current number syntax

## Summary

This RFC proposes aligning `tonumber` with Luau's current number syntax,
allowing for binary literals and underscores in number strings passed to `tonumber`.

## Motivation

Currently `tonumber` supports regular numbers and hexadecimals, making `tonumber` support half of luaus number syntax currently.
This is annoying for users as it means if they want to support `_` in numbers, if they are say making a math expression evaluator they cannot do so without doing some jank things with buffers and the bit32 library.
Same goes for binary literals, despite not being nearly as useful.

## Design

This RFC proposes allowing underscores in number strings given to `tonumber`, alongside allowing binary literals.
These changes make `tonumber` use the same syntax as luau for numbers:

```luau
print(tonumber("0b1111")) --> 15
print(tonumber("10_000")) --> 10000
```

## Drawbacks

This could in rare cases harm backwards compatibility, although very unlikely as if someone doesn't want to allow say underscores in numbers they most likely already have their own parser for whatever they are making.
