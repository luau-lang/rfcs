# Function parameter Names in User-Defined Type Functions

## Summary

This RFC proposes a method to add names to function parameters in user-defined type functions.

## Motivation

Any type that can be written by hand in Luau should also be able to be written programmatically in a user-defined type function. Functions with parameter names currently don't meet this criteria.

Furthermore, since parameter names currently aren't serialized in the type function runtime, *any* function type that goes through a user-defined type function loses that semantic information, which can be crucial to make the function self-describable.

Being able to add names to parameters defined in user-defined type functions increases the expressiveness of the type system, which is the primary motivation of the original [user-defined type functions RFC](https://github.com/luau-lang/rfcs/pull/45).

## Design

The following changes need to be made to the user-defined type functions API:

### `types.newfunction`

A parameter may be either a simple type (with no parameter name, like in the current behavior), or a table that contains both a parameter name and a parameter type.

The `head` field of the `parameters` parameter is now of type `{type | parameter}?`, where a `parameter` is defined like this:

```luau
type parameter = { name: string?, type: type }
```

Names that aren't strings, or that are invalid Luau identifiers, should be ignored.

```luau
function types.newfunction(parameters: { head: {type | parameter}?, tail: type? }, ...): type
```

### `functiontype:setparameters`

The `head` parameter is now of type `{type | parameter}?`, following the same rules as above.

```luau
function functiontype:setparameters(head: {type | parameter}?, tail: type?): ()
```

### `functiontype:parameters`

The `head` returned by this method is now of type `{parameter}?`, there is no `type` union variant because the table is normalized, such that parameter names are always represented by a `parameter` (unnamed parameters are *always* `{ name: nil, type: type }`, which avoids a branch).

This change breaks backwards compatibility, because current type functions assume that `functiontype:parameters().head` is a list of types. See the [Alternatives](#alternatives) section.

```luau
function functiontype:parameters(): { head: {parameter}?, tail: type? }
```

## Drawbacks

This proposal breaks backwards compatibility for the `functiontype:parameters()` method.

## Alternatives

Not doing this means users will continue to have problems where potentially important semantic information described in argument names will be lost using user-defined type functions. Furthermore, types that **need** to be created using user-defined type functions will not have parameter names, which makes creating self-describable functions harder.

The `functiontype:parameters()` method **can** be done in a backwards-compatible way (for example, by using SoA instead of AoS), but it generally results in code that is difficult to understand and easy to get wrong. This RFC proposes that breaking backwards compatibility is a drawback that is acceptable in this case, especially since user-defined type functions are relatively recent.
