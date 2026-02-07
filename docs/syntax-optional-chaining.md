# Summary

This RFC proposes adding an optional chaining operator (?.) to Luau to enable safe and concise access to nested table fields without requiring explicit nil checks.
Optional chaining short-circuits evaluation when the left-hand side is nil, returning nil instead of raising an error.

# Motivation

Accessing deeply nested tables in Luau often requires verbose nil-checking patterns, which reduce readability and are easy to get wrong.

For example,
```luau
local languageData =
    langName
    and lang[langName]
    and lang[langName].keys
```
or Luau-equivalent:

```luau
local languageData =
    if langName and lang[langName] and lang[langName].keys
    then lang[langName].keys
    else nil
```

# Design
With the optional chaining, the codes written above can be reduced to:
```luau
local languageData = lang[langName]?.keys
```

If lang[langName] evaluates to nil, the expression evaluates to nil without error.

# Backwards Compatibility
?. operator is already invalid in Lua and Luau, making backwards-compatibility safe to implement.

# Drawbacks
Optional chaining must respect existing __index and __newindex semantics.
