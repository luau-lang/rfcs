# NoInline Attribute for Functions

## Summary

This RFC proposes a function-level `@noinline` attribute to prevent individual functions from being inlined, even if they meet the required conditions.

## Motivation

Inlined functions will obviously not be present in stack traces, which can be annoying when debugging.

## Design

Syntactically, the `@noinline` attribute takes no parameters. It can be used on both top-level and inner functions. It does not apply recursively to the functions defined within the lexical scope of the attributed function. These "inner" functions have to be explicitly attributed for inline prevention.

In the following example, only `bar` will be inlined.

```lua
@noinline
local function foo()
    local function bar()
        return 42
    end
    return bar()
end

print(foo())
```

In this example, neither `foo` nor `bar` will be inlined.

```lua
@noinline
local function foo()
    @noinline
    local function bar()
        return 42
    end
    return bar()
end

print(foo())
```

## Drawbacks

Adding this attribute increases complexity of code.

## Alternatives

One alternative is to intentionally disqualify a function from the inline conditions, which means redundant code (use of getfenv/setfenv) or unnecessary/unwanted behavior (use of self or varargs in function parameters).
Another alternative is to explicitly lower the optimization level, such as via `--!optimize 1` which may kill desired optimizations.
