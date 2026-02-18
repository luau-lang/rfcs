# Resolve exported type of same name as the module implicitly.

## Summary

Remove redundancy when using an exported type that shares the same name as its module by implicitly resolving the type with the module's name.

## Motivation

It is common to export a type that represents what the module returns, especially in OOP. However, when using that type, you end up repeating the same name, ie: `Maid.Maid`, `Signal.Signal`, `Promise.Promise`.

This undoubtedly feels redundant and like boilerplate. While defining type aliases locally can mitigate this, they are also boilerplate.

## Design

When the name of a variable containing a module is `Something`, the type `Something` alone resolves to `Something.Something`, provided that:

- The type `Something` does not already exist locally or globally.
- The module exports a type named `Something`; otherwise, a type error is raised.

This is based on the name of the variable, not the name of the module's script or file.

For example:
```luau
local Something = require(path.to.Something)

local x: Something
```
Would be equivalent to:
```luau
local Something = require(path.to.Something)

local x: Something.Something
```

If a type alias of the same name is defined, it will immediately opaque the module, so doing `type Something = Something` results in a cyclic type.

This should be backward-compatible, intuitive, and straightforward to implement.

## Drawbacks

This requires the user to name their variables consistently, or if using the alternative outlined below, it would require the module to have the same name as the type it exports. However, since this is just syntactic sugar, the user is opting to use it.

## Alternatives

A different approach would be to use the module's actual name instead of the variable's name. So the following code would work:

```luau
local something = require(path.to.Something)

local x: Something
```

However this would be harder to implement, and the type system already uses the variable as the source for importing types, while this is disconnected from it, so the original design may fit better.