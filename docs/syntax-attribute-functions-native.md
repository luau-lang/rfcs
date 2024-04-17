# Native Attribute for Functions

## Summary

This RFC proposes a function-level `@native` attribute to request native compilation for individual functions, independent of the script-level `--!native` hotcomment.

## Motivation

Luau's native compiler currently compiles whole scripts annotated with `--!native` hotcomment. However, this provides very coarse-grained control. Since all functions in the script may not benefit from native compilation, developers might be forced to move unrelated functions together to natively compiled scripts. In this RFC, we propose a function-level `@native` attribute to facilitate developers to pick and choose individual functions for native compilation.

## Design

Syntactically, the `@native` attribute takes no parameters. It can be used on both top-level and inner functions. It applies recursively to all the functions defined within the lexical scope of the attributed function because the "inner" functions logically constitute the implementation of the attributed function. Hence, attributing "inner" functions as `@native` will not have any effect if an _ancestor_ function already has a `@native` attribute. 

In the following example, both `parent` and `child` will be natively compiled. 

```lua
@native
function parent()
    function child()
       -- do something
    end
    -- do something
end
```

On the other hand, in this example, only `child` will be natively compiled. 

```lua
function parent()
    @native
    function child()
       -- do something
    end
    -- do something
end
```

The implementation _may_ choose to issue warning in the following cases where `@native` attribute is redundant:

1. A function has more than one occurrence of `@native` attribute
2. An inner function has one or more occurrences of `@native` attribute when an _ancestor_ function already has a `@native` attribute.
3. A function has a `@native` attribute when the script is annotated with `--!native` hotcomment.

## Drawbacks

Introducing this attribute will have two adverse consequences:

1. It will increase the complexity of the implementation which will now have to make compilation decisions on a per-function basis.
2. Experience code will be strewn with occurrences of `@native` attribute.

## Alternatives

The alternative would be to not provide this attribute and rely on `--!native` hotcomment to make compilation decisions on a per-script basis. This might force developers to break their code organization and move unrelated functions together but it does not prevent them from getting performance benefits.