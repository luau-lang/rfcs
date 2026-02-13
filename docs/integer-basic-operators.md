# Basic Integer Operators

## Summary

Provide basic operators to Luau's new ``integer`` type. The basic operators described here are all signedness-agnostic to avoid any conflicts related to signedness.

## Motivation

Luau ``integer`` type does not support basic operators like ``+``, ``-`` or ``*`` despite all of these operators being fairly well-defined across all
uses of ``integer``'s regardless of interpretation of signedness or the ``integer``'s value itself. This makes operating with integers more difficult
and clunky (``integer.add(int1, int2)``). This RFC aims to implement these three basic operators without breaking backwards compatibility with the current
integer RFC to make using ``integer``s easier. The rational for not breaking backwards compatibility is given in the "Drawbacks" section.

## Design

Luau's integer type will be extended with the some basic operators, all of which have a direct analogue in the ``integer`` library. This ensures that the RFC remains minimal in scope. In particular, the below three operators are defined:

- ``integer + integer -> integer``

Returns ``integer.add(a, b)``.

- ``integer - integer -> integer``

Returns ``integer.sub(a, b)``

- ``integer * integer -> integer``

Returns ``integer.mul(a, b)``

Most notably, this RFC does not add operators like ``integer / integer`` which depends on signedness and should be covered in a separate RFC. 

Other methods like integer division etc. are also not specified by this RFC either to limit this RFC's scope and should also be covered in a separate RFC.

Lastly, this RFC does not implement operators which apply between a ``integer`` and a ``number``. This is due to numbers potentially having a fractional
part. The behaviour on how these should be resolved as well as how to implement these cross-type operators should also be covered in a separate RFC.

Example usage:

```luau
local a = 123i
local b = 124i
local c = 125i
print((a + b) * c - a) -- Prints (123 + 124) * 125 - 123
```

## Drawbacks

This RFC is very barebones and may itself be ambiguous despite the intentionally limited operator set chosen. Additionally, adding operators to a native type like this increases VM complexity, may worsen performance of existing types (like ``number``) and makes it impossible for embedders to define custom behaviour for these basic integer operators although this shouldn't be required in practice anyways.

Additionally, this RFC does not remove ``integer.add(a, b)`` from the ``integer`` library as doing so would be more complicated and would require changes to be made to an already approved RFC. This means that the ``integer`` standard library may have some "bloat" caused by having both ``a + b`` and ``integer.add(a, b)`` to add integers and vice versa for subtraction and multiplication. This design of merely adding the operators wholesale while keeping the ``integer.add`` methods was intentionally chosen to make this RFC smaller in scope. Additionally, there may be composability benefits to keeping ``integer.add`` as well.

## Alternatives

Do nothing. The existing ``integer`` library does already implement all of the above operators as functions and could be used instead. Additionally, a more comprehensive RFC that implements more integer operators could be merged instead.
