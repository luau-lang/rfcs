# Type Metamethod Support

## Summary

Add support for the `__type` metamethod on Luau defined tables.

## Motivation

The motivation of allowing developers to use the `__type` metamethod is to provide a better runtime mechanism for comparing between table types.

Right now, the type checker can compare table types, however, using typeof on these tables will still return `table`, as to the runtime, these are the same type. Support for the `__type` metamethod is already supported for host-defined userdatas, so it would be trivial to extend this to tables.

## Design

The design would simply involve adding a new metamethod `__type`, that can be set in the metatable of another table:

```lua
local Cat_MT = {
  __index = Cat,
  __type = "Cat"
}
```

When calling the `Cat` object with `typeof`, it should return the value of the defined `__type` metamethod if it exists:

```lua
local cat = Cat.new()
print(typeof(cat)) --> "Cat"
```
  
## Drawbacks

The main drawback is that this allows spoofing of host-defined userdatas, lets take the Roblox datum `Color3`. This object contains a `__type` flag of Color3. If we allow __type on userdatas, there's no easy way to check between a host-defined userdata, and a spoofed Color3 Luau userdata/table at the Luau level.

The easiest fix is to include a hardcoded prefix on the `__type` field to indicate its a Lua-defined table.

## Alternatives

This can already be implemented using a field exposed on the table:

```lua
local t1 = {__type = "Table1"}
local t2 = {__type = "Table2"}

local function useTable(t)
  if t.__type == "Table1" then
    print("table is of type 1")
  elseif t.__type = "Table2" then
    print("table is of type 2")
  end
end
```

The issue here is that it requires each table passed to the function to implement a `__type` field, which has its own problems

* If the table is not frozen (or cant be frozen), someone can easily change the `__type` field and make the code take a path other than expected.
* It creates a contract that every object table has a key of the same name to check types.
