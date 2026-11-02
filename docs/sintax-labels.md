# Luau Labels

## Summary

This proposal introduces named labels and `goto` statements to Luau, allowing explicit control-flow transfers within a program.

It also proposes treating labels as first-class references, allowing them to be stored in variables, passed to functions, returned from modules, and stored in tables.

The proposed syntax is:

```luau
label home

goto home
```

The `label` keyword is contextual, meaning it is interpreted as a label declaration only in the appropriate syntactic context.

The implementation may use either static or runtime label resolution, depending on the approach that best fits Luau's architecture.

## Motivation

Luau provides structured control-flow statements such as `if`, `while`, and `for`. While these cover most common use cases, they do not provide the same flexibility as explicit control-flow transfers.

Labels and `goto` would allow developers to define specific execution points and jump to them when necessary.

This could be useful for:

- Complex control-flow structures.
- State machines and execution flows.
- Low-level control-flow management.
- Situations where structured control flow becomes unnecessarily complicated.

Making labels first-class references would additionally allow developers to manipulate execution destinations dynamically.

## Design

### Label declarations

A label is declared using the `label` keyword followed by an identifier.

```luau
label home
```

The `label` keyword is contextual and does not need to become a globally reserved keyword.

### Goto statements

The `goto` statement transfers execution to a declared label.

```luau
label home

print("Hello")

goto home
```

The target may also be a label reference rather than a directly declared identifier.

```luau
local destination = home

goto destination
```

### Scope and visibility

Labels follow lexical scope visibility rules.

A label declared in an outer scope is accessible from its nested scopes.

```luau
label home

do
    goto home -- Valid
end
```

However, a label declared inside a nested scope is not accessible from its parent scope.

```luau
goto destination -- Error: label is not accessible

do
    label destination
end
```

The same restriction applies between sibling scopes.

A `goto` statement cannot target a label that is outside its accessible lexical scope.

### First-class label references

Labels can be treated as values and stored in variables.

```luau
label initialization

local destination = initialization

goto destination
```

They can also be passed to functions.

```luau
function jumpTo(destination: label)
    goto destination
end
```

Labels may be returned from functions.

```luau
function getLabel(): label
    label destination

    return destination
end

local destination = getLabel()
```

They may also be stored in tables.

```luau
label initialization

local destinations = {
    init = initialization
}

goto destinations.init
```

### Label lifetime

A label reference remains valid while it is retained by an existing reference.

For example:

```luau
local destination = getLabel()
```

The returned label is captured and retained by `destination`.

In contrast:

```luau
getLabel()
```

If the returned label is not captured or otherwise retained, it can cease to exist after the function returns.

This allows labels to have a lifetime associated with their references rather than necessarily being restricted to the lifetime of their original declaration scope.

### Module interaction

Labels may also be returned from modules and used by other modules.

For example:

```luau
-- here.luau

label destination

print("Hello, World from here.luau!")

return destination
```

```luau
-- main.luau

local destination = require("here.luau")

goto destination
```

This extends the use of labels beyond the module in which they were declared.

### Syntax comparison

Lua already supports labels and `goto`, but uses a different declaration syntax:

```lua
::home::

goto home
```

This proposal introduces an explicit declaration form:

```luau
label home

goto home
```

The proposed syntax also explores first-class label references, which are not provided by conventional Lua labels.

## Drawbacks

- Introducing labels adds another control-flow mechanism to the language.
- First-class label references require additional type-system considerations.
- Runtime label management may introduce implementation complexity.
- Label references require a well-defined lifetime and validity model.
- Explicit control-flow transfers can make code harder to understand when overused.
- Static analysis, debugging, and control-flow validation may become more complicated.

## Alternatives

### Structured control flow

Developers can continue using existing statements such as `if`, `while`, and `for`.

These constructs are generally preferable for ordinary control flow, but do not provide the same flexibility as labels and `goto`.

### Conventional labels

Another alternative is to introduce labels and `goto` without making labels first-class values.

This would provide explicit control-flow transfers while avoiding the additional complexity of dynamically retained label references.

### Lua-compatible syntax

Luau could adopt Lua's existing `::name::` syntax instead of introducing the `label` declaration form.

However, the proposed syntax provides a more explicit declaration and is designed to accommodate first-class label references.

## Prior Art

- **Lua:** Supports labels and `goto` using `::name::` declarations.
- **C and C++:** Support labels and `goto` for explicit control-flow transfers.
- **Luau:** Already provides structured control-flow statements, but does not currently provide the proposed label-reference model.
