# `const` bindings in for loops and function parameters

## Summary

Extend the existing `const` keyword to support per-binding const declarations in for loops variables and function parameters.

## Motivation

Luau already supports `const` for local bindings:

```luau
const x = 10
const function foo() end
```

However, there is no way to mark loop variables or function parameters as immutable. This leads to accidental reassignment bugs, especially in generic for loops where key/value pairs should remain unchanged, and in functions where parameters should be treated as read-only.

## Design

### New syntax

Generic loops:

```luau
for const k, const v in table do end   -- both const
for const k, v in table do end         -- only k const
for k, const v in table do end         -- only v const
```

Numeric loops:

```luau
for i = 1,10,1 do end -- normal usage
for const i = 1,10,1 do -- const variable i
    i *= 2 -- err
end
for const const = 1,10,1 do end -- const is constant variable --overwrite const keyword
```

Function parameters:
```luau
function foo(const x, const y) end     -- both const
function bar(const x, y) end           -- only x const
function baz(const, const x) end       -- const is not constant variable name --overwrite const keyword, x const
```

## Backward Compatibility

```luau
for const in x do -- const is not constant variable --overwrite const keyword -- If you used it this way before, there is no change.
end
    
for const const in x do -- const is constant variable --overwrite const keyword
end

for const = 1,10,1 do end -- if you used it this way before, there is no change

function foo(const) do -- const is not constant variable --overwrite const keyword -- If you used it this way before, there is no change.
end

function foo(const const) do -- const is constant variable name --overwrite const keyword
end
```

Rule: `const` is treated as a keyword **only when** the next token is a `Name` (and, in the `for` statement first-binding position, the token after that is not `=`))

## Drawbacks

- **Unusual syntax**: Normally, loops and parameter variables follow the pattern `name`, `name`, etc., but now you can add `const` before them.

## Alternatives

- **Do nothing**: Rely on linter rules or code conventions to catch accidental reassignment. This avoids parser complexity but provides no language-level guarantee.
