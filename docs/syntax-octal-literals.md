# Python-style octal literals

## Summary

This RFC proposes allowing octal literals to be defined in Luau in the form `0oNNN...` - for example, `0o777` evaluates to 511 in decimal.

## Motivation

Many other languages also support octal literals, so it allows for easier porting of code. 
Some legacy systems use octal extensively, and converting from octal to other bases to interface with them can be a chore.

## Design

Octal literals can be specified in Luau code with an `0o` prefix, just like how binary and hexadecimal literals use `0b` and `0x` respectively.

Identifiers beginning with `0o` are already illegal, so this should not impact any existing code.

## Drawbacks

Since they use only decimal digits, octal literals may appear confusing at first to other people reading code using them.

## Alternatives

Simply using other bases will of course work, however it can be time-consuming and in some cases there are meaningful advantages to using octal (e.g. PDP-11 instruction encodings).

Some languages use a leading zero to denote octal literals (most notably C), but this would break lots of existing code and is generally disliked by most programmers.
