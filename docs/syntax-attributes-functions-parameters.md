# Function Attribute Parameters

## Summary

This RFC proposes a syntax for function attribute parameters. This is a follow up to the [Attributes](./syntax-attributes-functions.md) RFC, which proposed a design for function attributes. We only propose a syntactic extension in this RFC; the rest of the implementation details remain unchanged.

## Motivation

The [Attributes](./syntax-attributes-functions.md) RFC provides a syntax for parameterless function attributes. This suffices for some potential use cases, such as `@native`, and `@inline`. However, other potential use cases such as `@deprecated` will benefit from an expressive attribute syntax that allows parameters to be supplied. We can use the parameters to provide the name of the function to be used in place of the deprecated function, and the reason for deprecation. This information can be used to generate informative deprecation warning messages. Another potential use case could be an `@unroll` attribute for loop unrolling which could use a numeric parameter for specifying the number of iterations to be unrolled. This would also require supporting attributes on loops, which would be a topic of discussion for a separate RFC. It might be desirable to allow these attributes to assume a default value for parameters not provided explicitly, but that's a discussion reserved for attribute-specific RFCs.

## Design

The following syntax is proposed for attributes with parameters:

```ebnf
attribute = '@' NAME 
          | '@' NAME '(' ')'
          | '@' NAME '(' ATOMIC-LITERAL (',' ATOMIC-LITERAL)* ')'
```

The extension proposed to the [Attributes](./syntax-attributes-functions.md) RFC is in the second and third lines which allow zero or more comma-separated literals to be supplied to the attribute. `NAME` should be a valid identifier immediately following `@` without any spaces. The `ATOMIC-LITERAL` category includes values of type `boolean`, `number`, `string`, and the `nil` value. The `nil` value can be be used for "optional" parameters when no valid value exists. As a hypothetical example, consider the `@deprecated(<new-function-name>, <warning-message>)` attribute. Assume a function is being marked deprecated because we are getting rid of the feature entirely and there is no other function replacing it. In that case, we can pass `nil` for the `<new-function-name>` parameter. An alternative would be to allow this attribute to have either one or two parameters. But that still won't help when an attribute can take three parameters and we want to ignore the second. A `(` following a `NAME`, with or without intervening whitespace, starts a parameter list for that attribute, even if the attribute does not support parameters. The parser will only check for syntacting correctness. The implemention will raise an error when it uses the attribute and finds that an invalid number or type of arguments were supplied. The implemention will treat an attribute with an empty parameter list equivalent to one without parameters. This means that `@native()` is the same as `@native`. This gives us forward compatibility if we decide to support attributes on parenthesized expressions in the future. Consider the following hypothetical example with a `@my_attribute` attribute, which does not take any parameters, applied to a parenthesized expression.

```lua
@my_attribute
(f().x)
```

Since an opening parenthesis starts a parameter list for the attribute, the expression `(f().x)` will be treated as a parameter list of `@my_attribute`. This will lead to a syntax error since attribute parameter can only be `ATOMIC-LITERAL` and `f().x` is not one. This ambiguity can be resolved by adding `()` to `@my_attribute` as follows:

```lua
@my_attribute()
(f().x)
```

There are two notable exclusions in this syntax.

1. Literal `table` values are excluded from being passed as parameters since they are composite structures; information supplied via table fields can be passed as separate parameters to the attribute. If they are still deemed useful, a follow-up RFC can introduce them with motivating use cases.
2. Named parameters are not supported. Our envisioned use cases rarely require parameters, and, since attributes are not user-extensible, we don't foresee use cases involving long parameter lists that would benefit from parameter names. Furthermore, named parameter lists introduce additional processing complexity by enabling parameters to be supplied in any order.

## Drawbacks

Extending attribute syntax to accept parameters contributes to the complexity of the language in the parsing phase.

## Alternatives

The alternative would be to not support attribute parameters. This will lead to substandard warning messages from `@deprecated` attribute. It would also prevent us from introducing `@unroll` attribute in the future.