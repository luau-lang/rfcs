# Function Attribute Parameters

## Summary

This RFC proposes a syntax for function attribute parameters. This is a follow up to the [Attributes](./syntax-attributes-functions.md) RFC, which proposed a design for function attributes. We only propose a syntactic extension in this RFC; the rest of the implementation details remain unchanged.

## Motivation

The [Attributes](./syntax-attributes-functions.md) RFC provides a syntax for parameterless function attributes. This suffices for some potential use cases, such as `@native`, and `@inline`. However, other potential cases such as `@deprecated` will benefit from an expressive attribute syntax that allows parameters to be supplied. We can use the parameters to provide the name of the function to be used in place of the deprecated function, and the reason for deprecation. This information can be used to generate informative deprecation warning messages. Another potential use case could be an `@unroll` attribute for loop unrolling which could use a numeric parameter for specifying the number of iterations to be unrolled. This would also require supporting attributes on loops, which would be a topic of discussion for a separate RFC. It might be desirable to allow these attributes to assume a default value for parameters not provided explicitly.

## Design

The following syntax is proposed for attributes:

```ebnf
parameter-table = '{' '}'
          | '{' parameter (sep parameter)* '}'

sep = ',' | ';'

parameter = literal 
          | NAME '=' literal

literal = BOOLEAN | NUMBER | STRING | NIL | parameter-table

list-attribute = NAME parameter-table
               | NAME

attribute = '@' NAME 
          | '@[' list-attribute (',' list-attribute)* ']'
```

In Luau scripts, attributes are specified before the `function` keyword in function definitions. In declaration files, attributes are specified in function type declarations, before the `function` keyword in `declare function (type, ...) : type` syntax, and before the `(` in the `name: (type, ...) : type` syntax. The primary extension proposed to the [Attributes](./syntax-attributes-functions.md) RFC is a new delimited syntax `@[]` for specifying multiple comma-separated attributes with parameters supplied through an optional Luau table. The important features are:

1. Inside the attribute list, `@[]`, attribute names are not allowed to have a leading `@`.
2. Attribute lists cannot be nested. Attributes cannot be specified on attribute parameters.
3. Attributes inside `@[]` are separated by a `,`.
4. Attributes inside `@[]` are allowed to take an optional table of parameters.
5. The "parameter table" is a Luau table whose entries specify attribute parameters, with or without names, separated by commas or semicolons.
6. A parameter can be any literal value: `BOOLEAN`, `NUMBER`, `STRING`, `NIL`,  or a `parameter-table`. Attributes can be seen as a way of providing tagged metadata with no evaluation semantics of their own. With that in mind, we only allow these syntactic categories because they evaluate to themselves. This is also the reason we disallow entries of the form `[exp1] = [exp2]` in parameter table, since this would require evaluating `exp1` at construction time. `nil` is allowed for the sake of generality and completeness.
7. Attributes can be specified before function declarations in multiple ways: `@attr1 @[attr2, attr3{2, "hi"}]`, `@attr1 @attr2 @[attr3{2, "hi"}]`, `@attr1 @[attr3{2, "hi"}] @attr2`, and  `@[attr1, attr2, attr3{2, "hi"}]` are all equivalent.
8. An attribute with an empty parameter list can be equivalently specified without one, i.e., `@[attr{}]` is same as `@attr`.

The parser is responsible for for enforcing the syntax specified by this RFC and ensuring that attributes are not repeated on a definition or a declaration. The number and type of parameters accepted by a particular attribute will be specified by its own RFC, and enforced by the relevant component of the implementation (typechecker, compiler, etc.), after parsing.

We currently have an ad-hoc implementation of `@checked` attribute, used in declaration files. It is specified in two ways:

1. `declare function @checked abs(n: number): number`
2. `writef64: @checked (b: buffer, offset: number, value: number) -> ()`

To ensure uniformity with the specification of this RFC, declarations using the first style will be modified to use `@checked` attribute before the `function` keyword. Declarations using the second style will remain unchanged.

An alternative parameter syntax would be to implement function-style attribute syntax of the form `@attr(parameters, ...)`, but that would not allow specifying parameter names. Attribute syntax of the form `@attr {...}` will create ambiguity when attributes without parameters are used on tables, should we decide to allow attributes on them in the future. An advantage of the `@[]` syntax is that it provides the `]` delimiter which marks the end of attribute, leaving open the possibility of syntax evolution without introducing parsing ambiguity.

## Drawbacks

Extending attribute syntax to accept parameters contributes to the complexity of the language in the parsing phase.

## Alternatives

The alternative would be to not support attribute parameters. This will lead to substandard warning messages from `@deprecated` attribute. It would also prevent us from introducing `@unroll` attribute in the future.
