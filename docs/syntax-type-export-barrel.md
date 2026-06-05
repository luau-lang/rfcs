# Export Type Barrel

## Summary

Add a top-level declaration that re-exports all names from another module’s exported type namespace.

```luau
export type * from "@self/Child"
```

This declaration makes every exported type from the target module available as an exported type of the current module, 
without creating a runtime binding and without affecting the module's returned value.

## Motivation

Today, exported types are accessed through the alias of a directly required module:

```luau
local Child = require("../pkgs/Interface/Child")

local a: Child.A
local b: Child.B
```

This works well when consumers can require the defining module directly. 
However, it becomes cumbersome when authors want to expose a flatter, 
package-level API surface through an interface or barrel module.

Consider a package structure like this:

```luau
-- roblox_packages/Interface/Child.luau
export type A<T = number, U = string> = ...
export type B = ...
export type C = ...

local Child = {}

return Child

-- roblox_packages/Interface/init.luau
local Child = require("@self/Child")

export type A<T = number, U = string> = Child.A<T, U>
export type B = Child.B
export type C = Child.C

return {
    Child = Child,
}

-- src/init.luau
local Interface = require("./roblox_packages/Interface")

local a: Interface.A<integer, 'Literal String'>
local b: Interface.B
local c: Interface.C
```

This pattern is repetitive, error-prone, and does not scale well when the exported type surface changes 
or when packages are layered through multiple interface modules.

There is a clear need for a way to re-export the exported types of another module without manually enumerating them one by one.

## Design

### Syntax

This RFC introduces one new top-level declaration form:

```ebnf
exportTypeReexport ::= 'export' 'type' '*' 'from' STRING
```

Example:

```luau
export type * from "@self/Child"
```

The module path must be a string literal and is resolved using the same rules as `require` paths.

### Semantics

`export type * from "path"` re-exports all exported type names from the target module into the exported type namespace of the current module.

Conceptually, this declaration behaves as though each exported type from the target module had been manually re-exported in the current module.

For example, if `"@self/Child"` exports the types `A`, `B`, and `C`, then:

```luau
export type * from "@self/Child"
```

is conceptually equivalent to:

```luau
local Child = require("@self/Child")

export type A = Child.A
export type B = Child.B
export type C = Child.C
```

except that the re-export does not require the module at runtime and does not introduce a runtime binding.

### Runtime impact

This declaration has no runtime effect.

- It does not create a runtime value binding.
- It does not modify the module's returned value.
- It does not imply value re-exports.
- It does not require the target module at runtime solely because of the re-export declaration.

### Name conflicts

If re-exporting introduces a duplicate exported type name, this is an error.

```luau
export type A = number
export type * from "@self/Child" -- error if Child also exports A
```

Likewise, if multiple `export type * from ...` declarations re-export the same type name into the same module, this is an error.

```luau
export type * from "@self/Left"
export type * from "@self/Right" -- error if both export A
```

A module with duplicate exported type names is invalid. Implementations may recover for tooling purposes, but the conflict remains an error.

### Interaction with module return values

As with existing `export type` declarations, type re-exports do not interfere with the module's runtime return value.

```luau
export type * from "@self/Child"

return {
    version = "1.0.0",
}
```

This is valid. The returned table controls the runtime API of the module. The re-export declaration only affects its exported type namespace.

### Interaction with cycles

This RFC does not introduce type-only cyclic imports, nor does it change any existing behavior around cyclic module structures.

It only provides a convenient way to forward already-exported types from one module into another module's exported type namespace.

## Drawbacks

### Tooling impact

This feature increases the complexity of editor tooling.

In particular, “go to definition” for a re-exported type may reasonably target either:
- the re-export site in the current module, or
- the original type definition in the source module.

Both behaviors are defensible, but tooling will need to choose one and present it consistently.

```luau
-- src/init.luau
local Interface = require("./roblox_packages/Interface")

local a: Interface.A -- luau-lsp navigates to `export type * from "@self/Child"` in Interface/init.luau to find the definition of A
local b: Interface.B -- luau-lsp navigates to `export type * from "@self/Child"` in Interface/init.luau to find the definition of B

-- roblox_packages/Interface/init.luau
export type * from "@self/Child" -- However, it is now unclear where A and B are actually defined. 

-- roblox_packages/Interface/Child/init.luau
export type * from "@self/Grandchild"
export type * from "@self/AnotherGrandchild"

-- roblox_packages/Interface/Child/Grandchild.luau
export type A = ...

-- roblox_packages/Interface/Child/AnotherGrandchild.luau
export type B = ...
```

This can be resolved by skipping the intermediate steps and tracking directly to the actual definition, but the ambiguity regarding the behavior remains.

Additionally, it becomes difficult to view the exported types at a glance within the interface.

### Limitations with generic composition

There are limitations to using `export type * from STRING`
in interface modules that combine generics to enable type-level cyclic references while avoiding runtime circular dependencies.

This necessitates either reverting to manually exporting each type one by one as before, or introducing new exceptional syntax. 
Alternatively, to handle exported type name conflicts, 
we could consider an edge case that makes an exception when a type is extracted via `*` and combined using generics. 
However, this approach would significantly increase complexity.

```luau
-- Interface/Child.luau
export type A<B = any> = ...

-- Interface/AnotherChild.luau
export type B<A = any> = ...

-- Interface/init.luau (error)
export type * from "@self/Child"
export type * from "@self/AnotherChild"

export type A = Child.A<AnotherChild.B> -- name conflict error
export type B = AnotherChild.B<Child.A> -- name conflict error
```

## Alternatives

### Continue using manual type aliases

The current approach is to re-export each type explicitly:

```luau
local Child = require("@self/Child")

export type A = Child.A
export type B = Child.B
export type C = Child.C
```

This works, but scales poorly and creates avoidable maintenance overhead.

### Add a broader import/export syntax

Another possible direction is a larger [import](https://github.com/luau-lang/rfcs/pull/103)/[export](https://github.com/luau-lang/rfcs/pull/179) RFC such as:

```luau
import type A, type B from "@self/Child"
export type A
export type B
```

However, a broader import/export syntax is a significantly larger feature. It should be considered separately.

This RFC instead focuses on a single, narrow use case: forwarding exported types from one module into another module's exported type namespace.

### Re-export a selected set of type names

A richer syntax could support named re-exports:

```luau
export type {A, B, C} from "@self/Child"
```

or renamed re-exports:

```luau
export type {A as ChildA} from "@self/Child"
```

A broader import/export redesign may eventually subsume this feature.
However, this proposal is intentionally smaller, independently useful, and does not need to wait on a larger module syntax overhaul.

### Re-export values and types together

Another alternative is a more general form such as:

```luau
export * from "@self/Child"
```

This is a much broader design space because it raises questions about 
runtime behavior, returned values, aliasing, binding semantics, evaluation order, and the interaction with any future value-export syntax. 
It is intentionally out of scope for this RFC.
