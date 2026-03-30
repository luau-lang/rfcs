# Table Comprehension

## Summary

Table comprehension is a specific type of syntax which allows you to construct tables using inline iteration and optional filtering expressions.

## Motivation

Constructing tables using loops is a very common practice in Luau, particularly for transforming and filtering collections. However, the current approach is verbose, even for simple transformations.

Examples:
```lua
-- Arrays
local numbers = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
local doubled = {}
for i, v in ipairs(numbers) do
    doubled[i] = v * 2
end

local evens = {}
for _, v in ipairs(numbers) do
    if v % 2 == 0 then
        table.insert(evens, v)
    end
end

-- Maps
local playersByName = {}
for _, player in ipairs(game.Players:GetPlayers()) do
    playersByName[player.Name] = player
end

local alivePlayers = {}
for _, player in ipairs(game.Players:GetPlayers()) do
    if player.Health > 0 then
        alivePlayers[player.UserId] = player
    end
end
```

Patterns like these are especially common in Roblox development, especially in cases such as building lookup tables and filtering instances by state.

However, this approach has the following drawbacks:
- Requires explicit mutation of tables (`table.insert`, index assignment)
- Introduces boilerplate for simple transformations
- Obscures intent by separating iteration, filtering, and assignment
- Increases the likelihood of indexing or insertion mistakes
- Lacks a concise expression form for common transformation patterns

Additionally, these patterns are difficult for the compiler to recognize as a single transformation, which limits opportunities for optimization.
A table comprehension syntax allows these operations to be expressed as a single construct. While it does not inherently change runtime behavior, it provides the compiler with a clearer representation of the programmer’s intent, which may enable future optimizations such as preallocation or loop specialization.

## Design

### Syntax

Two forms of table comprehension are proposed.

#### Array-style
`{expr for namelist in explist [if condition]}`

Examples:

```lua
local doubled = {v * 2 for i, v in ipairs(numbers)}
local evens = {v for _, v in ipairs(numbers) if v % 2 == 0}
```

#### Map-style
`{[key] = value for namelist in explist [if condition]}`

Examples:

```lua
local playersByName = {
    [player.Name] = player
    for _, player in ipairs(game.Players:GetPlayers())
}
local alivePlayers = {
    [player.UserId] = player
    for _, player in ipairs(game.Players:GetPlayers())
    if player.Health > 0
}
```

Whitespace and line breaks follow standard Lua rules and do not affect parsing. Both inline and multiline forms are valid.

Table comprehensions **cannot** be combined with standard table fields within the same constructor.

### Semantics

#### Evaluation order

- The iterator expression (`explist`) is evaluated once before iteration begins
- The comprehension executes with semantics equivalent to a `for` loop
- For each iteration:
    - Loop variables are assigned
    - The condition (if present) is evaluated
    - If the condition evaluates to a truthy value (or is absent):
      - The expression is evaluated
      - The result is inserted into the table

#### Scope

Variables introduced by the comprehension are scoped to the comprehension expression and are not visible outside of it, consistent with `for` loop semantics.

#### Array semantics

`{expr for namelist in explist}`

is equivalent to:

```lua
local result = {}
for namelist in explist do
    result[#result + 1] = expr
end
```

If `expr` evaluates to `nil`, the assignment behaves according to standard table semantics (`result[#result + 1] = nil`), which may result in holes in the array.

#### Map semantics

`{[key] = value for namelist in explist}`

is equivalent to:

```lua
local result = {}
for namelist in explist do
    result[key] = value
end
```

If duplicate keys are produced, the last assignment takes precedence.

If `value` evaluates to `nil`, the key is removed from the table, consistent with standard table behavior.

#### Conditional filters

If a condition is provided:

`{expr for namelist in explist if condition}`

it is equivalent to:

```lua
local result = {}
for namelist in explist do
    if condition then
        result[#result + 1] = expr
    end
end
```

For map-style comprehensions with a condition:

`{[key] = value for namelist in explist if condition}`

is equivalent to:

```lua
local result = {}
for namelist in explist do
    if condition then
        result[key] = value
    end
end
```

#### Iterator semantics

The `namelist in explist` portion follows the same semantics as Luau `for` loops, supporting all valid iterator forms including `ipairs`, `pairs`, and custom iterators.

### Grammar

## Drawbacks

Table comprehension introduces new syntax to the language, increasing overall language complexity. While the feature is designed to be minimal and consistent with existing `for` loop semantics, it adds another way to express iteration and table construction, which may impact readability for developers unfamiliar with the syntax.

Comprehensions may also be overused in complex scenarios, leading to code that is harder to read compared to explicit loops.

Additionally, this feature does not introduce new capabilities, as all functionality can already be expressed using existing `for` loops. Its primary benefit is improved expressiveness and conciseness.

## Alternatives

### Status quo (loop-based construction)

Continue using `for` loops to construct tables:

```lua
local result = {}
for _, v in ipairs(items) do
    if v > 1 then
        table.insert(result, v * 2)
    end
end
```

This approach is fully expressive and requires no new language features. However, it is more verbose and requires explicit mutation, which can obscure intent and introduce opportunities for errors.

### Helper functions

Users may define utility functions for common patterns such as mapping or filtering:

```lua
local function map(t, fn)
    local result = {}
    for i, v in ipairs(t) do
        result[i] = fn(v)
    end
    return result
end
```

While this reduces repetition, it introduces additional abstraction and may reduce clarity for simple transformations. It may also introduce overhead compared to inline constructs.
