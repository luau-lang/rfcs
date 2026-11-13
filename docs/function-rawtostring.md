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

The `__tostring` metamethod allows for the customization of how objects are represented as strings. However, sometimes it is useful to obtain the default string representation instead, like when debugging objects whose custom string representations obscure their identity. Currently, there is no straightforward way to obtain this representation without modifying the object's metatable, which is impossible for certain objects.

## Design

A new function, `rawtostring(obj: any): string`, will be added to the standard library. This function will behave almost identically to `tostring`, with the only difference being that it will not invoke the `__tostring` metamethod.

## Drawbacks

This introduces another global function to the standard library, adding a small amount of API surface and maintenance overhead.

## Alternatives

- Do nothing
- Add an optional second argument to `tostring` which would enable ignoring metamethods

## Prior Art

Luau already provides `rawequal`, `rawget`, `rawset`, and `rawlen`, which allow operations to bypass their respective metamethods.