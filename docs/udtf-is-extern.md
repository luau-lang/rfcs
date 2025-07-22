# ExternType's tag name

## Summary

Currently, the ``tag`` for an ExternType is ``class``. This RFC suggests to change it to ``extern``.

## Motivation

"class" is old and out-to-date, it treats ExternType as being a class, while it's not. It should be re-named to "extern" to avoid confusion.

The name is ideal, as it mirrors perfectly with the old syntax
``declare class`` and ``declare extern type``. Instead of ``class`` there's ``extern``. Making ``extern`` the suitable replacement for ``is("class")``.

## Design

This is the bulk of the proposal. Explain the design in enough detail for somebody familiar with the language to understand, and include examples of how the feature is used.

Instead of ``type:is("class")`` you'd do ``type:is("extern")``

### Changes

#### `type` Instance

| Instance Properties | Type |
| ------------- | ------------- |
| `tag` | `"nil" \| "unknown" \| "never" \| "any" \| "boolean" \| "number" \| "string" \| "singleton" \| "negation" \| "union" \| "intersection" \| "table" \| "function" \| "extern"` Remove: ~~`"class"`~~ |

## Drawbacks

It will break any code that currently uses ``is("class")``.

## Alternatives

### What is the impact of not doing this?

If we don't change this soon, it will standardize ``is("class")`` more and stay confusing. ``declare class`` was already changed to ``declare extern type`` syntax.
