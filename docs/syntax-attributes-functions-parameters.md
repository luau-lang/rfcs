# Function Attribute Parameters

## Summary

This RFC proposes a syntax for function attribute parameters. This is a follow up to the [Attributes](./syntax-attributes-functions.md) RFC, which proposed a design for function attributes. We only propose a syntactic extension in this RFC; the rest of the implementation details remain unchanged.

## Motivation

The [Attributes](./syntax-attributes-functions.md) RFC provides a syntax for parameterless function attributes. This suffices for some potential use cases, such as `@native`, and `@inline`. However, other potential cases such as `@deprecated` will benefit from an expressive attribute syntax that allows parameters to be supplied. We can use the parameters to provide the name of the function to be used in place of the deprecated function, and the reason for deprecation. This information can be used to generate informative deprecation warning messages. Another potential use case could be an `@unroll` attribute for loop unrolling which could use a numeric parameter for specifying the number of iterations to be unrolled. This would also require supporting attributes on loops, which would be a topic of discussion for a separate RFC.

## Design

The following syntax is proposed for attributes:

```ebnf
table ::= '{' [fieldlist] '}'
fieldlist ::= field {fieldsep field} [fieldsep]
field ::= Name '=' literal | literal 
fieldsep ::= ',' | ';'

literal ::= 'nil' | 'false' | 'true' | Number | String | table

parattr ::= NAME [literal]

attribute ::= '@' NAME | '@[' parattr {',' parattr} ']'

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

The primary extension proposed to the [Attributes](./syntax-attributes-functions.md) RFC is an attribute list, `@[]`, for specifying multiple comma-separated attributes with an optional literal parameter, and, an attribute syntax for declaration files.

The important syntactic features are:

1. Attribute lists cannot be empty; they should have at least one attribute.
2. Attributes inside attribute lists are separated by a `,`.
3. Attribute names cannot have a leading `@` inside attribute list.
4. Attribute lists cannot be nested.
5. Attributes cannot be used inside an attribute parameter.
6. Only attributes inside attribute lists can take a parameter, standalone attributes cannot.
7. An attribute parameter is a Luau literal: `true`, `false`, `nil`, `Number`, `String`, or a `table` composed of these literals with optional names. A trailing field separator is allowed.
8. Standalone and list style attributes can be mixed arbitrarily: `@attr1 @[attr2, attr3{2, "hi"}]`, `@attr1 @attr2 @[attr3{2, "hi"}]`, `@attr1 @[attr3{2, "hi"}] @attr2`, and  `@[attr1, attr2, attr3{2, "hi"}]` are all legal and equivalent.

Currently there is an ad-hoc implementation of `@checked` attribute, used in declaration files. It is specified in two ways:

1. `declare function @checked abs(n: number): number`
2. `writef64: @checked (b: buffer, offset: number, value: number) -> ()`

According to the syntax proposed in this RFC, the declarations using the first style will be modified to use `@checked` attribute before `declare` keyword. Declarations in the second style will remain unchanged.

Few important factors went in the design of this syntax:

1. Attributes are a mechanism for providing tagged metadata to the implementation. They are not user-extensible and have no evaluation semantics of their own. Hence, an attribute parameter can only be a literal, i.e., a value that evaluates to itself. This is also the reason we disallow entries of the form `[exp1] = [exp2]` in parameter table, since it would require evaluating `exp1` at construction time.
2. Function-style attribute syntax of the form `@attr({parameters})` were initially considered, but dropped becuase they do not allow us to specify parameter names. The optional table parameter proposed in the current syntax lets use specify multiple parameters as table entries, with and without names.
3. Using attributes with a table parameter outside attribute list, `@attr {...}`, will create ambiguity when used on tables, should we decide to allow attributes on arbitrary expressions in the future. Hence, the `@[]` delimited syntax was chosen to provide a `]` delimiter which marks the end of attributes, leaving open the possibility of syntax evolution without introducing parsing ambiguity.

The parser is responsible for for enforcing the syntax specified by this RFC and ensuring that attributes are not repeated on a definition or a declaration. The number and type of parameters accepted by a particular attribute will be specified by its own RFC, and enforced by the relevant component of the implementation (typechecker, compiler, etc.), after parsing.

## Drawbacks

Extending attribute syntax to accept parameters contributes to the complexity of the language in the parsing phase.

## Alternatives

The alternative would be to not support attribute parameters. This will lead to substandard warning messages from `@deprecated` attribute. It would also prevent us from introducing `@unroll` attribute in the future.
