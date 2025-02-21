# Support for thread and buffer Types in User-Defined Type Functions

**Status**: Implemented

## Summary

Add support for 'thread' and 'buffer' primitive types, omitted from original user-defined type function RFC.

## Motivation

While more rarely used than 'string' or 'number', 'thread' and 'buffer' are perfectly valid primitive types already supported by Luau typechecking engine, so should be included in user-defined type function API.

Developers are already hitting this limitation and there's no specific reason to exclude these types.

## Design

We propose two new library properties and two new type tags.

No methods are added for types with these new tags.

### Update `types` Library

| Library Properties | Type | Description |
| ------------- | ------------- | ------------- |
| `thread` | `type` | an immutable instance of the built-in type `thread` |
| `buffer` | `type` | an immutable instance of the built-in type `buffer` |

### Update `type` Instance

| Instance Properties | Type | Description |
| ------------- | ------------- | ------------- |
| `tag` | `"nil" \| "unknown" \| "never" \| "any" \| "boolean" \| "number" \| "string" \| "singleton" \| "negation" \| "union" \| "intersection" \| "table" \| "function" \| "class" \| "thread" \| "buffer"` | an immutable property holding this type's tag |

## Drawback

Unclear if this proposal has any drawbacks.
