# Native Attribute for Functions

**Status**: Implemented

## Summary

This RFC proposes a function-level `@native` attribute to request native compilation for individual functions, independent of the script-level `--!native` comment directive.

## Motivation

Luau's native compiler currently compiles whole scripts annotated with `--!native` comment directive. The compiler imposes an upper limit on the memory consumed by the generated native code which makes it important to target native compilation for functions that will benefit from it the most. This might force creators to break their code organization and move unrelated functions together to scripts marked `--!native`. In this RFC, we propose a function-level `@native` attribute to facilitate developers to request native compilation for individual functions. In the future, we want Luau's native compiler to automatically pick functions for native compilation, making the `--!native` comment directive redundant. Since compiler heuristics can be suboptimal, the proposed `@native` attribute would still remain useful by providing creators with a way to force native compilation of functions that were not automatically chosen by the compiler but would benefit significantly from native execution.

## Design

Syntactically, the `@native` attribute takes no parameters. It can be used on both top-level and inner functions. It does not apply recursively to the functions defined within the lexical scope of the attributed function. These "inner" functions have to be explicitly attributed for native compilation.

In the following example, only `parent` will be natively compiled.

```lua
@native
function parent()
    function child()
       -- do something
    end
    -- do something
end
```

On the other hand, in this example, both `parent` and `child` will be natively compiled.

```lua
@native
function parent()
    @native
    function child()
       -- do something
    end
    -- do something
end
```

## Drawbacks

Introducing this attribute will have two adverse consequences:

1. It will increase the complexity of the implementation which will now have to make compilation decisions on a per-function basis.
2. Experience code will be strewn with occurrences of `@native` attribute.

## Alternatives

The alternative would be to not provide this attribute and rely on `--!native` comment directive to make compilation decisions on a per-script basis. This might force developers to break their code organization and move unrelated functions together but it does not prevent them from getting performance benefits.
