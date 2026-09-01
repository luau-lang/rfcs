# `or return` Syntax

## Summary

The `or return` syntax is a more compact, understandable way to end execution if an expression evaluates to `nil`.

## Motivation

Existing ways of handling errors or `nil` values in Luau tend to be either verbose or dense. Code with nested `if` statements can make indents unnecessarily large, which hinders readability. Checks after variable initialization are better, but still make code lengthy and annoying to write.

`or return` would take this common idiom:
```lua
local var = foo()
if not var then return false end
```
and compact it into a single, simpler statement:
```lua
local var = foo() or return false
```

This idiom in various forms is omnipresent in nearly all Luau code, with prime examples occuring in Roblox development docs. The [second scripting tutorial](https://create.roblox.com/docs/tutorials/scripting/basic-scripting/deadly-lava) has this piece of code:
```lua
local lava = script.Parent

local function kill(otherPart)
    local partParent = otherPart.Parent
    local humanoid = partParent:FindFirstChild("Humanoid")
    if humanoid then
        humanoid.Health = 0
    end
end

lava.Touched:Connect(kill)
```
With this RFC implemented, the code could shrink to this, which I personally feel is more understandable at a glance:
```lua
local lava = script.Parent

local function kill(otherPart)
    local partParent = otherPart.Parent
    local humanoid = partParent:FindFirstChild("Humanoid") or return

    humanoid.Health = 0
end

lava.Touched:Connect(kill)
```

## Design

`or return` can be implemented as a binary operation with an optional right side, returning `nil` if no value is given.

The parsing step would be fairly simple as well. Since the new syntax is a way to express an already-supported feature more concisely, it can be transformed back into its original form in the parser. This means no changes are necessitated on the compiler side. If specified in the middle of an expression, it can temporarily store the value on its left side to perform the nil check, then compute the rest of the expression using the original copy. For example,
```lua
local var = bar(foo or return)
```
would be expanded into
```lua
local tmp = foo
if not foo then return end
local var = bar(foo)
```
which would preserve the existing behavior of expressions, while avoiding the need for compiler or AST changes.

## Drawbacks

Implementing this as an operation might make parsing harder, especially since `or return` overlaps with `or`. The optional right argument could also be an issue in more complex expressions, though giving the operation a low priority could help by nudging the user towards grouping statements with parentheses.

## Alternatives

There are already alternatives built into the language - this is purely a convenience feature. Regardless, the other options are more effort to write, which I believe means this shortcut option is worthy of consideration.
