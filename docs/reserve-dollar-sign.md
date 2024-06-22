# Reserve dollar sign (`$`) to be forever unused

## Summary

This RFC proposes a stable contract that `$` will not be used for any features in Luau, and will not be accepted as part of identifier tokens.

## Motivation

By reserving `$` external pattern matching tools, which at time of writing are already planned to be created by both Roblox and full-moon, can use it to symbolize things like placeholder tokens, similar to [ast-grep](https://ast-grep.github.io/guide/pattern-syntax.html). 

For example, a search in ast-grep for "checking equality to itself" would be `$A == $A`. This becomes ambiguous and harder to write if `$A` is also a valid identifier, or means some potential expression.

## Design

Nothing more than a stable contract is necessary. `$` will not be allowed in identifiers, and `$` will not be a component of any syntax being added to the language.

`$` is chosen as it seems fairly unlikely we will ever use it for any syntax, while at the same time its use as a placeholder token is common.
- ast-grep, as mentioned before
- SQL uses `$` for binded parameters in prepared statements

## Drawbacks

The dollar sign is used in a couple of places outside Luau:

- JavaScript, C++, and probably others allow it as an identifier, which jQuery uses as its primary controller
- PHP, Perl, Bash, and others both use it for variables (`$name = "John"`)
- Haskell uses it as an infix operator as sugar to help clean up function applications

These use cases are nowhere near compelling enough to where we would likely do the same.

## Alternatives

- Don't do it, as always.
- Pick a different sigil, though `$` is a fairly clear choice
