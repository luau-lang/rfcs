# switch for statements

## Summary

This RFC proposes a new `switch` syntax for the Luau language, adding `switch` statements for clearer, more concise branching logic.

## Motivation

The purpose of the `switch` statement is to simplify readability in cases where multiple branches depend on the value of one variable. Luau currently supports branching by using chains of `if`-`elseif`, but these may become wordy if multiple values are to be checked, or if fall-through behavior is desired. A `switch` syntax allows a simpler, easier to read structure.

## Design

The syntax below, now proposed, is using `switch value`, then the definition of cases with `for` blocks. Each case can take a tuple of comma-separated values; its code is inside `do` and `end`. A default case is a `do` block at the end.

Example:

```lua
switch value
    for "a" do
        -- Code for case "a"
    end
    for "b" do
        -- Code for case "b"
        break
    end
    for "a", "b", "c" do
        -- Code for any of these values
        break
    end
    do
-- Code for default case
    end
end
```

## Drawbacks

This syntax adds a pattern not found in Luau, and may necessitate concepts for developers to learn. The syntax looks like the syntax of `for` loops but isn't a loop, which can be confusing in some cases.

## Alternatives

For instance, in the absence of a syntax for `switch`, developers depend on `if`-`elseif` chains: In the absence of a syntax for `switch`, developers have to depend on `if`-`elseif` chains, which often become cumbersome and harder to read with multiple values to check. To that effect, consider handling the fall-through behavior of `switch` statements. Here every condition is explicitly repeated; thus, code becomes more verbose and error-prone:

```lua if value == "a" then
-- Code for case "a"
elseif value == "b" then
-- Code for case "b"
elseif value == "c" or value == "d" then
-- Code for case "c" or "d"
else
-- Default case
end
```

Using the above chained `if`s, the convenience a `switch` provides is consumed by repeated comparisons and explicit coding of fall-through behavior. This can result in longer branching code and is more annoying to maintain when cases are added or altered.
