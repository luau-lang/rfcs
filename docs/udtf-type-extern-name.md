# type ``externname`` method

## Summary

Add a new method for User Defined Type Functions called ``type.externname``. Which will retrieve a ``ExternType``'s name.

## Motivation

**Use cases:**
- A lazy _(due to it being string comparison, not structural check)_ to filter out extern types (e.g. from ``declare extern type``)
  - e.g. external types that are not accessible through ``types.@1`` such as ``vector`` for instance
  - embedders may also have their own defined external types that can't be filtered without passing a type into the function.
- Useful for debug purposes when using ``print`` within a type function.


Currently, you can only do this or other tricks:

```lua
--!strict
type function isType(input, whatToCheck)
  return (input == whatToCheck)
end

type function vectorOnly(input, whatToCheck)
  local matches = isType(input, whatToCheck)
  if (matches) then
    print(matches, "It is a vector type")
    return input
  end

  return types.unknown
end

type a = vectorOnly<typeof(vector.zero), vector>
```


**What it would solve:**
- You don't have to pass in a sample of a type that you want to check
- Drawback: If two ``ExternType`` are ever named the same, it would be an inaccurate check, hence why above it mentions _"lazy"_


## Design

Here's how an Unit Test for it would look like.

```lua
declare extern type CustomExternType with
    function testFunc(self): number
end

type function pass(arg, compare)
    if (arg:is("extern")) then
        assert(arg:externname() == compare:value())
    end

    return types.unknown
end

type a = pass<CustomExternType, "CustomExternType">
type b = pass<vector, "vector">
```


### ``type`` changes

| New/Update | Instance Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| New | `externname()` | `string?` | Returns the name of an ExternType or 'nil' if there's no name. |

**OR**

| New/Update | Instance Methods | Type | Description |
| ------------- | ------------- | ------------- | ------------- |
| Update | `name()` | `string?` | Returns the name of a generic or ExternType, or 'nil' if there's no name. |



### Example

```lua
declare extern type CustomExternType with
    function testFunc(self): number
end

type function onlyCustomExternType(input)
    assert(input:is("extern"))

    if (input:externname() == "CustomExternType") then
      -- ...
      return input
    else
      error("type is not named CustomExternType")
    end
end

type a = onlyCustomExternType<CustomExternType>
```


## Drawbacks

I don't know if a generic can be directly passed into a type function and then use ``type:name()``. But Generic Names are being collected.

If ``type:externname()`` would copy the ``externType->name`` into the TypeFunction's ExternType, maybe it would be inefficient for memory, because the name already exists.
In my first implementation, it doesn't do that though within the "serialize" process, instead it's just "read on request", but without caching.

It is currently confusing that there's ``:value()`` but the function name itself, doesn't speak out _"Hey, I am only for singletons"_ or similar.
Hence why I am wondering whether if something like this should exist, whether it should be ``:externname()`` or ``:name()``


## Alternatives

An alternative would be to have the ability to require "upvalue" or external types into the type function.
