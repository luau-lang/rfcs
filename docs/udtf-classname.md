# type classname method utility

## Summary

Add a new method to ``type`` for User Defined Type Functions, ``type.classname``. Which will retrieve a ``ExternType``'s name.

Or any other way to get the classname.

## Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

**Use cases:**
- A lazy (but therefore also quick way?) to filter out class based types (e.g. from ``declare class``)
  - e.g. class based types that are not accessible through ``types.`` such as ``vector`` for instance
  - embedders such as Roblox, may also have class based types that can't be filtered without passing a type into the function.
- Useful for debug purposes when using ``print`` within a type function.


Currently, you can only do this, or other tricks:

```lua
--!strict
type function isClass(input, whatToCheck)
  return (input == whatToCheck)
end

type function vectorOnly(input, whatToCheck)
  local matches = isClass(input, whatToCheck)
  if (matches) then
    print(matches, "It is a vector type")
    return input
  end

  return types.unknown
end

type a = vectorOnly<typeof(vector.zero), vector>
```


**What it would solve:**
- You don't have to pass in a sample of a type that you want to check, if it is a class
- Drawback: If two classes are ever named the same, it would be an inaccurate check, hence why above it mentions _"lazy"_


## Design

This is the bulk of the proposal. Explain the design in enough detail for somebody familiar with the language to understand, and include examples of how the feature is used.

I tried an implementation:
- https://github.com/karl-police/luau/commit/26b66c79d4d68e5f18601f619082b4d9b62473aa
- https://github.com/karl-police/luau/commit/03bef7130a9cdbf550b6e0bbcdaf5c9a562ef427

This is its **Unit Test**.

```lua
declare class CustomClass
    function testFunc(self): number
end

type function pass(arg, compare)
    if (arg:is("class")) then
        assert(arg:classname() == compare:value())
    end

    return types.unknown
end

type a = pass<CustomClass, "CustomClass">
type b = pass<vector, "vector">
```


### ``type`` changes

| New/Update | Instance Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New | `classname()` | `string?` | Returns the name of a class or 'nil' if there's no name. |

**OR**

| New/Update | Instance Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| Update | `name()` | `string?` | Returns the name of a generic or class, or 'nil' if there's no name. |



### Example

```lua
declare class CustomClass
    function testFunc(self): number
end

type function onlyClassName(input)
    assert(input:is("class"))

    if (input:classname() == "CustomClass") then
      -- ...
      return input
    else
      error("class is not named CustomClass")
    end
end

type a = onlyClassName<CustomClass>
```


## Drawbacks

I don't know if a generic can be directly passed into a type function and then use ``type:name()``.
But Generic Names are being collected.

If ``type:classname()`` would do this as well, maybe it would be inefficient for memory, because the name already exists.
In my first implementation, it doesn't do that though within the "serialize" process, instead it's just "read on request", but without caching.

It is currently confusing that there's ``:value()`` but the function name itself, doesn't speak out _"Hey, I am only for singletons"_ or similar.
Hence why I am wondering whether if something like this should exist, whether it should be ``:classname()`` or ``:name()``


## Alternatives

An alternative would be to have the ability to require "upvalue" or external types into the type function.
