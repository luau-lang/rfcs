# `extern` tag in User-Defined Type Functions.

## Summary

Currently, in user-defined type functions, the ``tag`` field for an external type is ``class``. This RFC proposes changing the value of the tag to ``extern``.

## Motivation

In the past, Luau type system internals played generally fast and loose with the terminology for types coming from our foreign function interface. We broadly referred to them
as class types because the types were known to be coming specifically from C++ in the context of Roblox, but Luau as a language supports embedding from C as well, and any
language that can speak a C ABI. To avoid confusion, especially for users curiously poking at some of the internals, we changed all the internal references from ``class`` to
"extern type" which is a more fitting name for what we actually know here: that the type describes some external or foreign data. The last remaining inconsistency here is
a user-facing one: we had mistakenly codified ``class`` as the name of the tag for extern types in user-defined type functions. We should correct this issue now, before
the New Type Solver moves into a released state and we are no longer able to change this for the sake of backwards compatibility.

## Design

The ``tag`` field of a type that is an external type will be ``"extern"`` instead of ``"class"``. To test a tag, you will write ``type:is("extern")`` instead of ``type:is("class")``.

### Changes

#### `type` Instance

| Instance Properties | Type |
| ------------- | ------------- |
| `tag` | `"nil" \| "unknown" \| "never" \| "any" \| "boolean" \| "number" \| "string" \| "singleton" \| "negation" \| "union" \| "intersection" \| "table" \| "function" \| "extern"` ~~`"class"`~~ |

## Drawbacks

The primary drawback of this is that some existing code using user-defined type functions in the New Type Solver Beta may be relying on testing that the tag is ``"class"``. The hope is that the affected surface of code here is currently very little because the API of user-defined type functions today does not allow a lot of useful operations to be performed on extern types (beyond exactly what is possible for tables), and we'd like to correct the issue if possible before the New Type Solver leaves beta, which would likely make breaking backwards compatibility more expensive.

## Alternatives

### What is the impact of not doing this?

The longer we wait to correct this, the harder it is to change, and we will have the persistent issue of user-defined type functions referring to "class types". Users will likely continue to be confused by this, and to continue to ask questions about how to employ these "class types" for object-oriented programming in Luau, which is not possible because they are actually foreign function interface types. This is the only place in user-facing syntax where Luau currently refers to "classes" at all as a concept, and without correcting this, we may run into issues down the line for any proposals that _do_ seek to add class-like constructs to the language in any capacity.
