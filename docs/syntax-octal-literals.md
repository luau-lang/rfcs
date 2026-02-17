# Octal integer literals

## Summary

This RFC proposes adding support for defining octal literals in the form `0oNNN...` - for example, `0o777` evaluates to 511 in decimal. 

## Motivation

Some legacy systems use octal extensively, and converting from octal to other bases to interface with them can be a chore. For example, the PDP-11 uses octal extensively - its instruction encodings are a natural fit for octal as they are arranged in groups of 3 bits, and octal is used to represent addresses & words in nearly all materials.

Unix also uses octal to represent file permissions (e.g. in commands like `chmod 755`), and this is perhaps the most recognisable use of it today.

Many other languages (Python, JavaScript, Rust...) also support octal literals in this style, so it allows for more straightforward porting of code.

The audience for this feature is admittedly small, but the parser provisions for supporting hexadecimal and binary literals extend easily onto adding octal literals.

## Design

The `0o` prefix was chosen mainly to line up with other programming languages. Python 3.0 was the first major language to implement it, and others such as JavaScript and Rust followed suit. It also aligns with how binary and hexadecimal literals use `0b` and `0x` respectively.

Identifiers beginning with `0o` are already otherwise illegal, so this should not impact any existing code.

In all ways other than base, they behave exactly like binary and hexadecimal literals, such as throwing errors on invalid digits.

## Drawbacks

Since they use only decimal digits and are somewhat uncommon, octal literals may appear confusing at first to other people reading code using them.

## Alternatives

Simply using other bases will of course work, however it can be time-consuming and as seen above there can be meaningful advantages to using octal. 

Some languages use a leading zero to denote octal literals (most notably C and languages derived from it), but this would break lots of existing code and is generally considered an annoyance by most programmers.
