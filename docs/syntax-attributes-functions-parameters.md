# Function Attribute Parameters

**Status**: Implemented

## Summary

This RFC proposes a syntax for function attribute parameters. This is a follow up to the [Attributes](./syntax-attributes-functions.md) RFC, which proposed a design for function attributes. We only propose a syntactic extension in this RFC; the rest of the implementation details remain unchanged.

## Motivation

The [Attributes](./syntax-attributes-functions.md) RFC provides a syntax for parameterless function attributes. This suffices for some potential use cases, such as `@native`, and `@inline`. However, other potential cases such as `@deprecated` will benefit from an expressive attribute syntax that allows parameters to be supplied. We can use the parameters to provide the name of the function to be used in place of the deprecated function, and the reason for deprecation. This information can be used to generate informative deprecation warning messages. Another potential use case could be an `@unroll` attribute for loop unrolling which could use a numeric parameter for specifying the number of iterations to be unrolled. This would also require supporting attributes on loops, which would be a topic of discussion for a separate RFC.

## Design

Attributes with parameters can only be specified inside `@[]`, separated by commas. Attribute parameter syntax mirrors the syntax for function calls; parameters are specified as a comma-separated list of zero or more literal values enclosed in parentheses, which are optional for single-parameter attributes if the parameter is either a literal string or a table constructor. `@[]` cannot be empty or nested. Attributes inside `@[]` cannot have a leading `@`. Attribute parameters cannot have attributes on them.

`@[attr "literal string"]`, `@[attr {key = 1, 2, 3}]`, `@[attr()]`, `@[attr(1, 2, 3)]`, `@[attr(1, nil, true, false, "literal string", {name = "value", 3.0})]` are some examples of attributes that conform to the proposed syntax.

The formal syntax for attributes with parameters is described below.

```ebnf
littable ::= '{' [fieldlist] '}'
fieldlist ::= field {fieldsep field} [fieldsep]
field ::= Name '=' literal | literal 
fieldsep ::= ',' | ';'

literal ::= 'nil' | 'false' | 'true' | Numeral | LiteralString | littable

litlist ::= literal {‘,’ literal}

pars ::= ‘(’ [litlist] ‘)’ | littable | LiteralString 

parattr ::= Name [pars]

attribute ::= '@' Name | '@[' parattr {',' parattr} ']'

attributes ::= {attribute}
```

Attributes can appear before `function` and `local function` statements, augmenting the allowed statement syntax as follows:

```ebnf
stat ::= attributes 'function' funcname funcbody
stat ::= attributes 'local' 'function' Name funcbody
```

Attributes can also used in declaration files with the following syntax:

```ebnf
type ::= Type | Name ':' Type
decl ::= attributes 'declare' 'function' '(' {type} ')' : Type
decl ::= 'declare' Name ':' '{' {Name ':' attributes '(' {type} ')' '->' Type} '}'
```

The extensions proposed to the [Attributes](./syntax-attributes-functions.md) RFC are:

1. An attribute list, `@[]`, for specifying multiple comma-separated attributes with parameters.
2. A parameter syntax for attributes.
3. An attribute syntax for declaration files.

Attributes with parameters are enclosed with `@[]` to facilitate syntax evolution without introducing parsing ambibguity. Without `@[]`, if we decide to allow attributes on arbitrary expressions in the future, `@attr(...)` will have two meanings; an attribute with parameters or an attribute on a parenthesized expression. `@[]` resolves this by providing a closing delimiter, `]`, to separate the attribute from the expression. `@[` has been chosen instead of `[` since it is unique and does not introduce ambiguity if we enable attributes on table entries of the form `[exp1] = [exp2]`.

`@attr`, `@[attr]`, and `@[attr()]` are equivalent. The order in which attributes is specified is irrelevant. Hence, `@attr1 @[attr2, attr3(2, "hi")]`, `@attr1 @attr2 @[attr3(2, "hi")]`, `@attr1 @[attr3(2, "hi")] @attr2`, and  `@[attr1, attr2, attr3(2, "hi")]` are also equivalent.

Attribute parameters are restricted to be literals, i.e., values that evaluate to themselves. This is because attributes are a mechanism for providing tagged metadata to the implementation. They are not user-extensible and have no evaluation semantics of their own. For this reason, even entries of the form `[exp1] = [exp2]` in table literal parameters are disallowed, since they would require evaluating `exp1` at construction time.

Attributes should not be repeated on a definition or a declaration and every attribute should receive the correct number and type of parameters. Non-conformance should result in a parsing error, obviating the need for a separate linting pass.

## Drawbacks

Extending attribute syntax to accept parameters contributes to the complexity of the language in the parsing phase.

## Alternatives

The alternative is to not support attribute parameters. This will lead to substandard warning messages from `@deprecated` attribute. It will also prevent us from introducing `@unroll` attribute in the future.
