# syntax-nullish-coalescing-assignment

## Summary

Add a `??=` compound assignment operator that assigns to a variable only when its current value is `nil`.

## Motivation

Assigning a default value to a variable only when it is `nil` is one of the most common patterns in Luau: config table initialization, player data defaults, optional argument handling, and lazy caching all use it constantly. There is no concise and correct way to express this today.

The shortest available shorthand is `or`:

```lua
config.Enabled = config.Enabled or true
```

This works when the existing value is always truthy or nil, but it checks truthiness rather than nil-ness. A legitimate `false` value gets silently overwritten, making it unsafe for boolean flags and any other case where `false` is a meaningful value.

The correct form requires an explicit nil check:

```lua
config.Enabled = config.Enabled == nil and true or config.Enabled
```

This is correct but requires writing the left-hand side path twice, which gets increasingly painful with longer paths like `playerData[userId].settings.volume`. The explicit `if` form avoids the double write but takes three lines for what is conceptually a one-liner.

## Design

`??=` is a compound assignment operator. The statement:

```lua
x ??= y
```

is equivalent to:

```lua
if x == nil then
    x = y
end
```

The right-hand side is only evaluated if the left-hand side is `nil`, consistent with short-circuit evaluation. Like the existing compound assignment operators, `??=` is a statement, not an expression, and supports a single value on each side.

```lua
-- correct but verbose (3 lines)
if config.Volume == nil then
    config.Volume = 1
end

-- correct but requires writing the path twice
config.Volume = config.Volume == nil and 1 or config.Volume

-- with ??=
config.Volume ??= 1
```

For boolean values, `or` is not a safe substitute:

```lua
-- broken: overwrites false
config.Enabled = config.Enabled or true

-- with ??=: correctly preserves false
config.Enabled ??= true
```

## Drawbacks

Introduces new syntax. Developers unfamiliar with nullish coalescing from other languages will need to learn what `??=` means and how it differs from `or`.

## Alternatives

The `or` shorthand covers the nil assignment case only when `false` is not a meaningful value. The explicit nil check form is always correct but verbose and requires writing the variable path twice. The explicit `if` form is correct but takes three lines. None of these are a satisfying substitute for a first-class operator. Not doing this leaves a gap in the compound assignment operator set that developers work around daily with boilerplate.
