# Syntax Attribute for Silencing Warnings

## Summary

Introduce a syntax attribute (`@nolint`) for expressions and statements to silence editor warnings.

## Motivation

Users today may have many lints enabled in their projects. With [`lute lint`](https://lute.luau.org/cli/lint/), the
number is likely to only grow. `--!nolint` hot comments do not allow users to granularly disable warnings in specific
areas of their code. Adding standardized syntax for silencing in-editor warnings improves the granularity and the
visibility of this feature.

## Design

The `@nolint` attribute will be allowed preceding function bodies, `do end` blocks, `for` / `while` / `repeat` loops,
variable assignment, types, type aliases, and type functions. Inside the following syntax node, editor warnings will be
silenced by default.

Warnings derived from those things (e.g., if a function is deprecated and has `@nolint`), will not be silenced. To be
more specific, the `@nolint` attribute should only silence warnings if they are fully enclosed by the syntax node
`@nolint` is attached to.

`@nolint` will also have optional attribute parameters allowing users to exclude specific lint names with either string
literals (`@[nolint "LocalUnused"]`), or lists of string literals (`@[nolint("LocalUnused", "MisleadingCondition")]`).
This is very coherent with the existing top-level comment `--!nolint LocalShadow`.

For now, pattern matching, message matching, et cetera are out of scope for this RFC, because they
would be significantly more complex and leave wild room for tools to vary in implementation without
much immediately obvious benefit.

## Drawbacks

- This augments attribute syntax, so it comes with natural costs to updating parsers / tooling and integrating the
feature with existing software.

- People implementing lint tooling for Luau will need to actively consider `@nolint` and interpret
parameters to exclude named lints, but this should be fine because most linters already require luau
parsers to function

- People may be quick to jump to using `@nolint` for important lints. However, similarly to things like `--!nocheck`,
that is their perogative and is better than disabling it for an entire file or in a config scope.

- People writing code to be compatible with lua & luau would not be able to silence lints.

## Alternatives

- `@lint` / `@[lint { name = true } ]`. This is more coherent with `.config.luau` files, but most cases for in-line lint
silencing only need to specify which lints they are looking for, values of `true` seem like they'd become repetitive
when compared to a declaration with a larger scope, are cumbersome to type, and don't allow people to easily mask all
lints as they could with `@nolint`.

- Name it `nowarn` instead. This is disconnected from the file-wide flag `--!nolint`, which could be interpreted as a
boon or a flaw. However, users might also conflate `nowarn` with runtime warnings, and we want to avoid confusing
people.

- Use a comment instead. The only benefit to this is compatibility when running in a lua runtime, which seems moot at
this point as many luau features today (and from the start) are exclusive with lua's syntax expectations.

- Do nothing, which means linter projects will need to invent their own syntax or hot comments for this (e.g.,
the formatter project StyLua relies on `--stylua: ignore` / `--stylua: ignore start`). Each project may have different
syntax or rules, which is a double edged sword.

- Explicit pattern matching syntax following a standard (e.g., lua(u) string patterns). The complexity hit from this
seems like it wouldn't be worth it.
