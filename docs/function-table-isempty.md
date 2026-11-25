# Add table.isempty() to the standard library

## Summary

This RFC proposes adding `table.isempty(t)` to Luau's standard library, providing a readable and explicit way to determine whether a table contains any entries.

## Motivation

Currently, checking whether a table is empty typically requires the following expression:

```luau
if next(t) == nil then
    -- Table is empty
end
```

While this approach is concise and efficient, it relies on an iteration primitive to express an emptiness check.

Developers reading this expression must understand the behavior of `next()`, rather than being able to immediately recognize the intent of the condition.

The proposed function would allow developers to write:

```luau
if table.isempty(t) then
    -- Table is empty
end
```

This makes the intent explicit, improving readability and discoverability without introducing new language syntax.

The function would work with arrays, dictionaries, and mixed tables, avoiding confusion with the length operator (`#`), which is not a reliable emptiness check for arbitrary tables.

## Design

The proposed function has the following signature:

```luau
function table.isempty(t: table): boolean
```

It returns `true` if the provided table has no entries, and `false` otherwise.

Its behavior is equivalent to `next(t) == nil`.

Examples:

```luau
table.isempty({}) -- true
table.isempty({1, 2, 3}) -- false
table.isempty({name = "Luau"}) -- false

local sparse = {}
sparse[100] = true

table.isempty(sparse) -- false

local t = setmetatable({}, {
    __index = function()
        return 123
    end,
})

table.isempty(t) -- true
```

The function would not modify the table, invoke metamethods, or depend on its sequence length.

It would support frozen tables and reject non-table arguments.

No new syntax or special type inference rules would be required.

## Drawbacks

The proposed functionality is already available through `next(t) == nil`, so it does not introduce new capabilities.

Adding a standard library function also increases the API surface and its long-term maintenance cost.

The proposal does not claim any performance improvement over the existing approach.

Its primary benefit is readability, which must be weighed against the cost of introducing another standard library function.

## Alternatives

### Continue using next()

The existing approach is concise, efficient, and familiar to experienced Lua developers.

However, it does not explicitly communicate the intended operation.

### Use the length operator

`#t == 0` can be used with sequences but does not reliably detect whether arbitrary dictionaries or sparse tables contain entries.

### Implement a user-defined helper

Developers can implement the function themselves:

```luau
local function isEmpty(t: table): boolean
    return next(t) == nil
end
```

This avoids expanding the standard library but requires individual projects to provide their own utilities.

## Prior Art

- Rust provides an `is_empty()` method for collections such as `Vec<T>`.
- OpenResty's LuaJIT fork provides `table.isempty()`.
- The Luau ecosystem also includes utility libraries such as RbxUtil, which provides `TableUtil.IsEmpty()`.

These implementations demonstrate that explicitly named emptiness checks are an established API pattern.

The proposed function would bring a similar interface to Luau's standard table library.
