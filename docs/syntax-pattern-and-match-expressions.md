# Pattern and match expression syntax

## Summary

This RFC proposes the introduction of pattern and match expression syntax to Luau, providing a powerful way to match values resulting in more readable, less verbose code.

## Motivation

The purpose of this RFC is twofold:

- Improve code readability and reduce verbosity; and
- Increase developers efficiency and move closer to being on-par with other programming languages.

An extremely good use case for match expressions is a parser, where different nodes need to be parsed depending on the current token kind. Take the following code snippet for example:

```luau
local function parse_simple_expr(): AstExprNode
    if current_token.kind == "string" then
        return parse_string_expr()
    elseif current_token.kind == "number" then
        return parse_number_expr()
    elseif current_token.kind == "true" or current_token.kind == "false" then
        return parse_boolean_expr()
    else
        return error(`unexpected token {current_token.kind} "{current_token.text}" {current_token.span.x}:{current_token.span.y}`)
    end
end
```

The important information (such as whether `current_token.kind` is `"string"`, `"number"`, `"true"`, or `"false"`) isn't unreadable, but is hidden under a layer of extremely repetitive and verbose `if` statements. With the syntax proposed, this code can be simplified significantly:

```luau
local function parse_simple_expr(): AstExprNode
    return for current_token.kind match (
        "string" -> parse_string_expr(),
        "number" -> parse_number_expr(),
        "true" or "false" -> parse_boolean_expr(),
        * -> error(`unexpected token {current_token.kind} "{current_token.text}" {current_token.span.x}:{current_token.span.y}`),
    )
end
```

The match expression distils the most important parts of the first example but removes almost all repetition and verbosity.

## Design

There are two main components proposed in this RFC:

- patterns, which check if a value *matches* some definition; and
- match expressions, which match one value to a collection of arms and return a consequence of the arm if it's pattern matches.

The proposed grammar changes are below:

```ebnf
patternname = NAME | NAME '[' exp ']' | patternname '.' NAME
pattern = NUMBER | STRING | 'nil' | 'true' | 'false' | stringinterp | patternname | '*' | pattern 'or' pattern | '(' pattern ')' | NUMBER 'until' NUMBER | 'not' pattern
matcharm = pattern ['if' exp] '->' exp
matcharmlist = matcharm {',' matcharm} [',']
matchexp = 'for' exp 'match' '(' matcharmlist ')'

simpleexp = NUMBER | STRING | 'nil' | 'true' | 'false' | '...' | tableconstructor | attributes 'function' funcbody | prefixexp | ifelseexp | stringinterp | matchexp
```

This syntax is heavily inspired by [Rust's match expression](https://doc.rust-lang.org/reference/expressions/match-expr.html) and [pattern syntax](https://doc.rust-lang.org/reference/patterns.html).

### Patterns

Patterns are the most powerful part of this RFC and without them match expressions would be next to useless. The syntax is designed to be extensible and flexible enough for use in other areas of the language in future.

The main purpose of a pattern is to check if a value *matches* some definition.

#### Exact

The *exact* pattern matches exactly what value is given to it. A pattern is an exact if given the following:

- Strings and interpolated strings;
- Numbers;
- Booleans; and finally
- Identifiers and index expressions.
Anything else produces a syntax error.

> Call expressions were considered as an addition to this list, however, they close the door to future syntax and introduce complexity with regards to multi-returns - all with little to no actual gain.
>
> A simple solution to this problem for the user is to move the call into a variable above and use an exact pattern with the new variable.

> *Discuss in comments*: Disallowing expression-based indexing was considered with the benefit of keeping the door opening to future syntax. However, it was decided that it could be confusing that only one form of indexing is allowed and it would also prevent users from accessing specific keys which use reserved keywords.

Some examples of exact patterns would be:

- `4`
- `"hello"`
- `foo.bar`
- `true`

Exact patterns are equivalent to a binary expression checking that value `==` the exact value. This means that `4` is equivalent to `value == 4`.

#### Wildcard

The *wildcard* pattern always matches, whatever the value. It is denoted with the asterisk (`*`).
> *Discuss in comments*: `_` and `else` were considered as alternatives to the `*` when denoting wildcard patterns.
>
> `_` was discarded as it is a valid identifier and so is ambiguous with exact patterns.
> `else` was discarded simply because it is more verbose than the other two. However, the benefit of explicitness (by using `else`) may overrule brevity.

Wildcard patterns are always equivalent to `true`.

#### Group

The *group* pattern matches if the pattern inside it matches as well. It is denoted by a pattern, wrapped in parentheses.

An example of a wildcard pattern would be `(4)`.

#### Or

The *or* pattern matches if either the left or right pattern matches. It is denoted with the `or` keyword in the infix position which takes a pattern on it's left and right.
> *Discuss in comments*: The `|` symbol was considered as an alternative to the `or` keyword when denoting or patterns but was discarded as it is usually denotes a Boolean OR. However, the benefit of brevity may overrule the concerns about `|` meaning Boolean OR not logical OR.

An example of an or pattern would be:

```luau
2 or "hello"
```

Or patterns are equivalent to every sub-pattern's binary expression, each wrapped in parentheses, with an `or` in between them. This means that `2 or 4` is equivalent to `(value == 2) or (value == 4)`.

#### Not

The not pattern matches if the pattern to the right of it doesn't match. It is denoted with the `not` keyword followed by a pattern.
> *Discuss in comments*: The `~` symbol was considered as an alternative to the `not` keyword when denoting not patterns but was discarded due to the fact that `not` is already used to mean logical NOT. However, the benefit of brevity (by using `~`) may overrule the concerns of a new symbol being added.
>
> In addition, it could be confusing that the not pattern acts sightly differently to the `not` unary operator. This is because the not pattern means "anything but the given pattern matches" and the `not` unary expression means "anything that is falsy is true (or matches)".

Not patterns are equivalent to `not` with the pattern on the right binary expression wrapped in parentheses. This means that `not nil` is equivalent to `not (value == nil)`.

#### Range

The *range* pattern matches if the value is a number and is inclusively within the bounds of min and max. A range pattern is denoted with an `until` keyword in the infix position, which takes a number on it's left and right, those being the min and max.
> *Discuss in comments*: `..` was considered as an alternative to the `until` keyword when denoting range pattern but was discarded as they already perform a completely different operation (concatenation) which could be confusing. However, the benefit of brevity (by using `..`) might overrule the concerns of confusion.

An example of a range pattern would be:

```luau
13 until 19
```

Range patterns are equivalent to a type check that asserts the value is a number, a `>=` check for min and a `<=` check for max. This means that `13 until 19` is equivalent to the binary expressions `type(value) == "number" and value >= 13 and value <= 19`.

#### Future: Structure
>
> *This RFC does not formally propose this syntax as it requires acceptance of a currently pending proposal. The following syntax is purely hypothetical and would need to be finalized in a separate RFC. It is only included here for completeness.*

The structure pattern is a superset of the proposed [Structure matching syntax](https://github.com/luau-lang/rfcs/pull/95). The pattern matches if the value is a table and fits the defined structure. In the context of match expressions, the defined keys are also bound to the scope of both the guard and the consequence.

The ability to add a pattern match for a specific key can be done by adding a `:` to the end of any key. `not nil` is the default pattern for a key when one is not specified.

An example of a match expression that uses structure patterns would be:

```luau
local result = do_thing_that_results()
local data = for result match (
    {
        .ok: true,
        .value,
    } -> do_something_with_value(value),
    {
        .ok: false,
        .message,
    } -> error(`thing resulted in an error with message "{message}"`)
)
```

#### Future: Multi-value
>
> *This RFC does not formally propose the following syntax as it is seen to be out of scope and would add additional complexity to match expressions and the proposal as a whole.*

A multi-value pattern matches a set of values with a set of patterns. It is denoted like the group pattern by being wrapped in parentheses, only this time more than one pattern is allowed, each being separated by a comma.

An example of a multi-value pattern would be:

```luau
(*, 13 until 19, "hello")
```

A value can also be bound to the pattern's current scope by adding `=` and a variable name after the pattern, like so:

```luau
local data = for pcall(do_fallible_thing) match (
    (true, * = data) -> data,
    (false, * = message) -> error(`fallible thing failed with message "{message}"`)
)
```

In the context of match expressions, they would be bound to the scope of both the guard and the consequence.

Multi-value patterns are equivalent to each pattern wrapped in parentheses and separated by `and`. This means that `(*, 13 until 19, "hello")` is equivalent to `(true) and (type(value2) == "number" and value2 >= 13 and value2 <= 19) and (value3 == "hello")`.

### Match expression

A *match* is a valid expression and consists of two parts:

- A *value* to compare each match arm against.
- One or more *match arm*s to check.
They are denoted with the `for` keyword, followed by an expression, the contextual keyword `match`, and finally the match arms wrapped in parentheses.

> *Discuss in comments:* `in` was considered as an alternative to `for` when denoting the start of a match expression but was discarded as it reads like you're looking *into* (e.g. via table access) a value, not at it. However, it could be argued that `for` causes more confusion as it is usually the start of a for loop.
>
> The `end` keyword was also considered as an alternative to the parentheses but was discarded as it reads worse when a match expression is inline.
>
> Finally, it was considered whether `match` should have the ability to take more than one value and introduce [multi-value patterns](#future-multi-value). This idea was purposely left out of the current proposal as it adds complications due to Luau not having tuples as first-class citizens. However, this does not mean it can't be added in a future RFC.

The first pattern matching, guard passing arm is evaluated and returned as the value of the match expression; otherwise, if no arms matched, the returned value is `nil`.

An example of a simple match expression would be:

```luau
local sides = for shape match (
    "line" -> 1,
    "triangle" -> 3,
    "square" or "rectangle" -> 4,
    "circle" -> error("circles don't have sides"),
    * -> error(`unknown shape {shape}`),
)
```

#### Match arms

A match arm (or arm for short) consists of three parts:

- A *pattern* to match the value against.
- An optional *guard*.
- A *consequence* expression.
They are denoted with a pattern, an optional `if` keyword and expression, followed by the `->` symbol, and closed with the consequence expression.

> *Discuss in comments*: A suggested alternative for the `->` symbol was the `then` keyword, however, this idea was discarded over verbosity concerns and the fact that it didn't read correctly.

An example of a simple match arm would be:

```luau
"triangle" -> 3
```

#### Guard

A *guard* is an optional check which is performed after an arm's pattern matches but before it's consequence is evaluated. It is positioned after an arm's pattern and is denoted with the `if` keyword, followed by the additional expression to check.

An example of a match arm with a guard would be:

```luau
"pudding" if likes_chocolate -> "chocolate"
```

## Drawbacks

The introduction of match expression and pattern syntax:

- increases parser and grammar complexity;
- expands the scope of what developers need to learn and therefore may be confusing; and
- may not be forwards compatible.

## Alternatives

### Don't do anything

The classic, always available option. As Luau already supports both statement and expression variants of `if`s, the proposed syntax could be considered purely syntactic sugar and ultimately an unnecessary addition to the language.

However, there seems to a general consensus that a `match` syntax should be introduced, whatever they look like. Additionally, with the inclusion of patterns, it opens the door to an extremely powerful feature which currently available syntax (such as `if`s) simply does not provide, making this proposal arguably much more than just syntactical sugar.

### Using a switch statement

There was [an RFC that proposed adding switch statements](https://github.com/luau-lang/rfcs/pull/63) to Luau which could be seen as a sensible alternative to `match` expressions because they are similar in scope. However, the RFC was rejected with a general consensus that switch statements shouldn't be added to Luau - especially with the syntax and semantics that were proposed.
