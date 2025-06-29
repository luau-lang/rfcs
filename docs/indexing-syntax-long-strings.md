# Indexing syntax when using long strings

## Summary
This RFC defines the parsing logic for `foo[[[a]]]` and `{[[[a]]]=b}`, making this both a parsable statement and removing currently semantic whitespace.

## Motivation
Luau supports multiline strings by means of long brackets. These take the form `[[foo]]`, `[=[foo]=]` and so forth. These are typically consumed during lexical analysis of implementations, before parsing has begun.

Luau additionally supports indexing, using square brackets. This can be used as part of an expression (`foo[a]`) or as part of a table literal (`{[a] = foo}`). This is identified during parsing, after lexical analysis has identified a single `[` opening token.

These two rules become conflicting when attempting to parse a statement of the form `foo[[[a]]]`. No strict behaviour is defined for how this should be parsed, however current implementations consume the first `[[` during lexical analysis, producing a string of `[a]]` followed by a single `]` token. This then fails to parse.

By rewriting this expression as `foo[ [[a]]]` we can ensure the lexical analysis identifies a single `[` followed by a string and then a closing `]`. This whitespace is therefore semantic.

This behaviour is unexpected, as there is a valid way this expression could have been parsed that was not used.

This is currently one of two semantic whitespace occurrences in the language. The other occurrence occurs when using a comment after a hanging minus symbol, as demonstrated in the following example:

```lua
local foo = 5 - -- Foo!
                1
```

This RFC does not address this second case, as the solution to this case is far less trivial.

### Existing behaviour across tooling
The predominant tool used for minification of Luau code at the time of writing, [darklua](https://github.com/seaofvoices/darklua), does not fall into the trap of minifying `foo[ [[a]]]` into `foo[[[a]]]`. It instead minifies to `foo[ [[a]] ]` or `foo['a']` depending on the configuration. Darklua would be able to remove those spaces after this change.

The first party Luau parser follows the semantic-whitespace behaviour.

The predominant Rust parser for Luau, [full-moon](https://github.com/Kampfkarren/full-moon), follows the semantic-whitespace behaviour.

## Design
A sequence of three opening square brackets (`[[[`) must be parsed as a single opening bracket followed by an opening long bracket (`'[' '[['`).

A sequence of two opening square brackets, followed by an equals symbol (`[[=`) must be parsed as a single opening bracket followed by the beginning of an opening long bracket (`'[' '[==...=[')`).

## Drawbacks
All current Luau parser implementations do not follow this behaviour. They instead follow the behaviour outlined in the Motivation section.

Under the current behaviour `foo[bar[[[a]]]` parses as `foo[bar"[a"]`, however this proposed behaviour would parse that as `foo[bar["a"]` with an unterminated bracket pair. This is not backwards compatible, however the author of this RFC believes this to be a more expected behaviour, and could not find evidence of this problematic pattern being used in the wild.

Under the current behaviour `print([[[[a]])` parses as `print("[[a")`. The change proposed here would cause this to parse as `print(["[a")` which is now invalid code. A single use of this was found on GitHub in a Lua file used for a custom nvim init.

## Alternatives
There are two alternatives:

1. Do absolutely nothing. This is undesirable, as this behaviour remains ambiguous
2. Retain the current behaviour, defining it explicitly with another RFC

Of these, (2.) would be preferable if the proposed behaviour in this RFC is rejected.
