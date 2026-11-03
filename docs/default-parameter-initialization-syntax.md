# Parameter Default Initialization Syntax

## Summary

This RFC proposes a syntax change which allows users to bind default values to function parameters without sacrificing code readability or type support. This allows developers to allocate default values
to function parameters without writing another line and chaining an `or` condition, which takes up 2 lines and makes the code look ugly. It would also introduce type recognition for that parameter, which
you can't get even by chaining an `or` condition. This does not collide with pre-existing syntax because the `=` token is unparsable in function parameters anyway.

## Motivation

Many developers often forget to initialize the parameters by default, which can raise errors at runtime.
This happens due to the developers being lazy or not wanting to make their code look bad by introducing redefinitions.

## Design

At it's core, it's a very simple feature; Be **able** to bind *every* parameter with a default value.
This accounts for closure parameters such as `a, b, c` in `function(a, b, c) end` and `d, e, f` in `local function foo(d, e, f)`.
It extends the default parsing for function parameters from a declaration to a binding. It starts with the normal parameter declaration e.g `a: b` and then checks for an extended `=` token.
If that token is present, it parses the parameter as a default parameter initialization "statement". The parameter would get evaluated on each function call, preserving similar behaviour to the
desugared variant (see Alternatives).


An example of this syntax would be:
```luau
local function createPath(
    name = "DefaultPathName",
    parent: Instance = workspace,
    transparent: boolean = true,
    position: Vector3 = Vector3.zero
): Part
    local path = Instance.new("Part")

    path.Size = Vector3.new(1, 1, 1)
    path.Position = position
    path.Parent = parent
    path.Transparency = transparent and 1 or 0
    path.Name = name

    return path
end
```



## Drawbacks

It would complicate typechecking due to a dilemma about type assumption: Do we or do we not assume the type based on it's default initializer value?
Take `function(foo = 1) end` as an example. If we assume the type, it would be `foo: number`, but how do we know that this function is never invoked again
with a different parameter type, in a seperate module, or using `getfenv`? This complicates it and then we turn to *not* making it typed, but this would
contradict Luau's behaviour, because `local a = 1` makes `a`'s type definitely `number`. Also, when desugared, the type warps back to `unknown` because `x` is `unknown`
and `x = x or 0` evaluates `x` to `unknown | number` which is `unknown`. The logical best choice would be to assign it it's type, to keep up with Luau's syntactic behaviour.

## Alternatives

The most common and used alternative is just adding a redefinition right after the function declaration like so:

```luau
local function pixel(x, y)
    x = x or 0
    y = y or 0

    return { size = { 1, 1 }, x = x, y = y }
end
```

which works just as fine as the feature proposed.

## Prior Art

Python already has this feature, as well as C++ and other high-level languages.

Python has a strikingly similar implementation to what I'm doing:

```py
def greet(name = "World"):
    print(f"Hello, {name}!")
```

but Python evaluates the expression only once on script start, whereas this RFC aims to evaluate it every function call.
