# Require Top-Level Annotations in Strict Mode

## Summary

Warn strict-mode developers when analysis encounters an unannotated top-level function.

## Motivation

In most programming languages featuring type inference, it is considered good practice to annotate top-level symbols.

It's a good idea for a few reasons:

### Library Contracts

First, top-level annotations are already a very good idea for any library author.  Without them, any change to any function might inadvertently change its type in a way that could break downstream consumers of the library.

### Structural Type Inference Gets Pretty Complicated

Structural type inference of OO APIs tends to get complicated even in simple scenarios.  Consider the following fragment:

```luau
local function isCharacter(inst): boolean
    local isBasePart = inst:IsA("BasePart")
    local result = false
    if not isBasePart then
        result = inst:FindFirstChildOfClass("Humanoid")
            and inst:FindFirstChild("HumanoidRootPart")
    end
    return result
end

-- Intentionally trigger a type error
local a: never = isCharacter
```

We produce the following error for this program:

```
Type Error: (12,18) Expected this to be unreachable, but got
    '<a, b, c, d..., e..., f...>(t1) -> boolean where t1 = { read FindFirstChild: (t1, string) -> (c, f...), ... 2 more ... }'
```

If we hand-format the type of `isCharacter`, it looks like this:

```
<a, b, c, d..., e..., f...>(t1) -> boolean
  where
    t1 = {
        read FindFirstChild: (t1, string) -> (c, f...),
        read FindFirstChildOfClass: (t1, string) -> (b, e...),
        read IsA: (t1, string) -> (a, d...)
    }
```

This is quite a lot\!  These long types are far too complex to understand when they appear in an error message or a tooltip.  A simple `: Instance` annotation on the type of `inst` makes things much simpler to understand.  It also greatly increases the quality of our autocomplete system.

The benefits of top-level annotations tend to trickle downward into the program: top-level types act as seeds that help us to infer the interior of each function with much higher fidelity.

## Design

In strict mode, we add a simple extra check when we encounter functions.

We will warn on missing annotations on a function if it is declared using a function statement (eg `function foo`) or if it is a method or field of a class.

We will *not* warn on unannotated parameters for lambdas or functions that appear within nested scopes.

We will also not warn on a missing return type annotation if inference indicates that a function returns 0 values.

Note that we leave developers with a simple way to explicitly opt out if they would truly prefer not to annotate something:

```luau
export const myFunction = function(arg, arg, arg)
    ...
end
```

Luau already offers capable type inference, so it's very easy for Luau to include the inferred type of the function in the warning message.  This makes it straightforward for the developer (or their text editor) to fill in the needed annotation.

A future improvement would be to extrapolate a suggestion from the inferred type.  For instance, if Luau witnesses an inferred type along the lines of `{ read find: ..., read gsub: ... }`, inference could suggest `string`.

## Drawbacks

The obvious one is that we're asking developers to put a bit of extra work in.  To shore this up, we'd like to offer editor tooling to automatically fill type annotations in for the user.

## Alternatives

The most compelling alternative here is frankly to do nothing.

Luau is already pretty good at inferring function argument types.  It isn't always able to infer the most intuitive type, but it could fairly be argued that it does well enough.
