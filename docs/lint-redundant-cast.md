# New lint rule: `RedundantCast`

## Summary

This RFC proposes a new, 29th lint rule that emits a warning upon using **typecasting** on variables where the solver had already inferred the same types. This change aims to encourage cleaner & more idiomatic code that is less prone to runtime errors. This change should apply with both the current solver as well as the new one.

## Motivation

The Luau solver is able to infer variables with high confidence. Despite this, some users choose to assert every type explicitly, either out of habit, preference for explicitness, or to document intent more clearly. This can lead to redundant type assertions that may mislead people unfamiliar with Luau’s type semantics, and in some poorly handled cases, cause runtime errors.

This RFC also serves as a cleanup for redundant casts that were originally written to bandage old type solver issues.

## Design

The rule operates by checking **3** conditions:
- A variable's type is assigned using the casting operator `::`
- The variable's type was not set to `any`
- The solver's inferred type is non-generic and is equal to the casted type

```luau
--!nolint LocalUnused

local x: number = 1 -- ✅
local y: string = "hello" -- ✅

local function returnAny(): any
    return "hello"
end

local w = returnAny() :: string -- ✅

-- Solver is already confident about type "z", use a type annotation to silence
local z = 2 :: number -- ❌

local function returnRandom(): any
    return (math.random(1, 2) == 1) and "hello" or 2
end

-- No warning is emitted, the type of "a" is cast to any
local a = returnRandom() :: any 
```


## Drawbacks

Some developers may find this rule too opinionated or stylistic, especially for those who favor explicit type assertions for documentation or consistency.

This lint rule aims at a coding oversight which isn't that common amongst developers, nor has immediate significant impacts if left ignored.

## Alternatives

- Setting this linting rule to be **disabled** by default
- Don't do it and disregard this RFC
