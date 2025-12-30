# Align `tonumber` with current number syntax

## Summary

This RFC proposes aligning `tonumber` with Luau's current number syntax,
allowing for binary literals and underscores in number strings passed to `tonumber`.

## Motivation

Currently `tonumber` supports regular numbers and hexadecimals, via the `base` argument and adding `0x` as a prefix to the string being passed to `tonumber`.
Given `tonumber` supports hexadecimal via `0x`:
```luau
print(tonumnber("0xFFFFFF")) --> 16777215
```

Its currently odd that it doesn't support binary literals via `0b` like luau currently does for numbers.
Additionally allowing underscores in numbers passed to `tonumber` is fairly easy to implement, and is nice syntax sugar for making parsers that handle numbers in luau.
Alongside the fact allowing underscores would also make the behavior of `tonumber` simpler for users. As `tonumber` would then parse numbers in the exact same way as Luau,
rather than how it is now with an odd caveat for hexadecimals.

## Design

This RFC proposes allowing underscores in number strings given to `tonumber`, alongside allowing binary literals.
These changes make `tonumber` use the same syntax as luau for numbers:

```luau
print(tonumber("0b1111")) --> 15
print(tonumber("10_000")) --> 10000
```

## Drawbacks

This could in rare cases harm backwards compatibility, although very unlikely as if someone doesn't want to allow say underscores in numbers they most likely already have their own parser for whatever they are making.
