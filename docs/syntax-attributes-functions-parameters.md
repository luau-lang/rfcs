# Function Attribute Parameters

## Summary

This RFC proposes a syntax for function attribute parameters. This is a follow up to the [Attributes](./syntax-attributes-functions.md) RFC, which proposed a design for function attributes. We only propose a syntactic extension in this RFC; the rest of the implementation details remain unchanged.

## Motivation

The [Attributes](./syntax-attributes-functions.md) RFC provides a syntax for parameterless function attributes. This suffices for some potential use cases, such as `@native`, and `@inline`. However, other potential use cases such as `@deprecated` will benefit from an expressive attribute syntax that allows parameters to be supplied. We can use the parameters to provide the name of the function to be used in place of the deprecated function, and the reason for deprecation. This information can be used to generate informative deprecation warning messages. Another potential use case could be an `@unroll` attribute for loop unrolling which could use a numeric parameter for specifying the number of iterations to be unrolled. This would also require supporting attributes on loops, which would be a topic of discussion for a separate RFC. It might be desirable to allow these attributes to assume a default value for parameters not provided explicitly, but that's a discussion reserved for attribute-specific RFCs. 

## Design

The following syntax is proposed for attributes with parameters:

```ebnf
attribute = '@' NAME 
          | '@' NAME '(' ATOMIC-LITERAL (',' ATOMIC-LITERAL)* ')'
```

The only extension proposed to the [Attributes](./syntax-attributes-functions.md) RFC is in the second line which allows one or more comma-separated literals to be supplied to the attribute. The `ATOMIC-LITERAL` category includes values of type `boolean`, `number`, `string`, and the `nil` value. The `nil` value can be be used for "optional" parameters when no valid value exists. As a hypothetical example, consider the `@deprecated(<new-function-name>, <warning-message>)` attribute. Assume a function is being marked deprecated because we are getting rid of the feature entirely and there is no other function replacing it. In that case, we can pass `nil` for the `<new-function-name>` parameter. An alternative would be to allow this attribute to have either one or two parameters. But that still won't help when an attribute can take three parameters and we want to ignore the second.

There are three notable exclusions in this syntax. 

1. Literal `table` values are excluded from being passed as parameters since they are composite structures; information supplied via table fields can be passed as separate parameters to the attribute. If they are still deemed useful, a follow-up RFC can introduce them with motivating use cases.
2. Attributes with empty parameter lists are disallowed. It can be argued that this leads to syntactic irregularity, but attributes with empty parameter lists convey no useful information over parameterless attributes, and have no compelling use case. Furthermore, they might complicate attribute processing in the implementation which will have to check for both `@NAME` and `@NAME()` forms and treat them equally.
3. Named parameters are not supported. Our envisioned use cases rarely require parameters, and, since attributes are not user-extensible, we don't foresee use cases involving long parameter lists that would benefit from parameter names. Furthermore, named parameter lists introduce additional processing complexity by enabling parameters to be supplied in any order.

## Drawbacks

Extending attribute syntax to accept parameters contributes to the complexity of the language in the parsing phase.

## Alternatives

The alternative would be to not support attribute parameters. This will lead to substandard warning messages from `@deprecated` attribute. It would also prevent us from introducing `@unroll` attribute in the future.