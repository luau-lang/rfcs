# `rawtostring`

## Summary

Adds a new `rawtostring` function to the standard library which, in converting a value to a string, bypasses the `__tostring` metamethod.

```luau
const object = setmetatable({}, {
    __tostring = function()
        return "Foo"
    end
})

print(tostring(object))    -- Foo
print(rawtostring(object)) -- default table representation (table: 0x...)
```

## Motivation

The `__tostring` metamethod allows for the customization of how objects are represented as strings. However, sometimes it is useful to obtain the default string representation instead. Currently, there is no straightforward way to obtain this representation without modifying the object's metatable, which is impossible for certain objects.

One such situation is in implementing `__tostring` metamethods, where you may want to provide additional context about an object while retaining its underlying identity. For example, a Promise implementation could use `rawtostring` to provide a more descriptive representation without losing the object's original identity:

```luau
function Promise:__tostring(): string
    return "Promise (" .. rawtostring(self) .. ")"
end
```

Another use case is debugging and logging. Custom string representations can obscure an object's underlying identity, particularly when working with opaque objects or objects whose metatables cannot be modified. Having access to the default representation makes it possible to inspect these objects without relying on their custom representations.

Additionally, custom `__tostring` implementations are user-defined code and may produce unexpected results, perform expensive operations, or raise errors. Diagnostic tools may need a way to obtain a representation of a value without invoking potentially problematic metamethods.

`rawtostring` provides a simple way to obtain this representation while preserving the behavior of `tostring`.

## Design

A new function, `rawtostring(obj: any): string`, will be added to the standard library. This function will behave almost identically to `tostring`, with the only difference being that it will not invoke the `__tostring` metamethod.

## Drawbacks

This introduces another global function to the standard library, adding a small amount of API surface and maintenance overhead.

## Alternatives

- Do nothing
- Add an optional second argument to `tostring` which would enable ignoring metamethods
- Add `rawtostring` to the `debug` library instead of exposing it globally

Adding `rawtostring` to the `debug` library is worth considering, since debugging is one of the primary use cases. However, `rawtostring` is also useful in ordinary runtime code, particularly when implementing custom `__tostring` metamethods. Furthermore, the function follows the same convention as the existing global `rawequal`, `rawget`, `rawset`, and `rawlen` functions: providing an operation that bypasses a particular metamethod.

## Prior Art

Luau already provides `rawequal`, `rawget`, `rawset`, and `rawlen`, which allow operations to bypass their respective metamethods.