# Leading `|` and `&` in types

**Status**: Implemented

## Summary

Allow the use of `|` and `&` without any types preceding them.

## Motivation

Occasionally, you will have many different types in a union or intersection type that exceeds a reasonable line limit and end up formatting it to avoid horizontal scrolling. Using the English alphabet as an example:

```luau
type EnglishAlphabet = "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z" | "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M" | "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z"
```

Or you might just format it for readability:

```luau
type EnglishAlphabet = never
    | "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m"
    | "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z"
    | "A" | "B" | "C" | "D" | "E" | "F" | "G" | "H" | "I" | "J" | "K" | "L" | "M"
    | "N" | "O" | "P" | "Q" | "R" | "S" | "T" | "U" | "V" | "W" | "X" | "Y" | "Z"
```

Currently, there are two solutions to effect it:

1. Moving `=` to the next line
2. Keep `=` on the line and add `never` if using `|`, or `unknown` if using `&`

```luau
-- 1) union:
type Result<T, E>
    = { tag: "ok", value: T }
    | { tag: "err", error: E }

-- 2) union:
type Result<T, E> = never
    | { tag: "ok", value: T }
    | { tag: "err", error: E }

-- 1) intersection:
type MyOverloadedFunction
    = ((string) -> number)
    & ((number) -> string)

-- 2) intersection:
type MyOverloadedFunction = unknown
    & ((string) -> number)
    & ((number) -> string)
```

OCaml and TypeScript support this same syntax:

```ocaml
type 'a tree =
  | Leaf
  | Node of 'a tree * 'a * 'a tree;;
```

```ts
type Tree<T> =
    | { type: "leaf" }
    | { type: "node", left: Tree<T>, value: T, right: Tree<T> };
```

## Design

This type becomes valid Luau syntax.

```luau
type Tree<T> =
    | { type: "leaf" }
    | { type: "node", left: Tree<T>, value: T, right: Tree<T> }
```

## Drawbacks

No drawbacks. The parser change is trivial. Instead of trying to parse a simple type and then calling `Parser::parseTypeSuffix`, simply sense whether the current lexeme is `|` or `&` and directly invoke `Parser::parseTypeSuffix` with `nullptr` for the first type and making sure that `nullptr` does not get pushed into the resulting vector. If the current lexeme is not `|` or `&`, then revert to current logic.

## Alternatives

Don't do this and accept one of the solutions already mentioned.
