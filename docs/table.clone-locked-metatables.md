# Feature name

## Summary

Allow `table.clone` to copy tables with locked metatables.

## Motivation

As proposed by the [`table.clone` RFC](function-table-clone.md), the function `table.clone` cannot create shallow copies of a table if it has a locked metatable.
Due to this ugly limitation, it is extremely un-ergonomic to create shallow copies of arbitary tables.
The current behavior of using `table.clone` on a table with a locked metatable is a hard-fail approach where `table.clone` spits out an error instead of having a soft-fail approach wherein a clone is generated without a metatable.
This hard-fail approach severely hinders the usefulness of `table.clone` due to in practice users having to re-implement the function in many scenarios.

## Design

When `table.clone` attempts to clone a table with a locked metatable the table is shallow copied with the exception of not assigning the metatable to the shallow copy.

ie. The behavior (as described by Lua pseudocode) would go from:

```lua
function table.clone(t)
assert(type(t) == "table")
local nt = {}

for k, v in pairs(t) do
nt[k] =v
end

if type(getmetatable(t)) == "table" then
setmetatable(nt, getmetatable(t))
end

return nt
end
```

to:

```lua
function table.clone(t)
assert(type(t) == "table")
local nt = {}

for k, v in pairs(t) do
nt[k] =v
end

local mtLocked = getmetatable(t) ~= nil and not pcall(setmetatable, t, getmetatable(t))

if not mtLocked and type(getmetatable(t)) == "table" then
setmetatable(nt, getmetatable(t))
end

return nt
end
```

## Drawbacks

There are no drawbacks to this.
The original RFC puts forth a rationale for restricting cloning tables with locked metatables due to security reasons.
This is a valid concern. The issue at hand being an avenue to assign metatables to unintended tables.
However, this is not a concern as the changes proposed in this RFC do not allow such behavior.

## Alternatives

An alternate mode of action would be to also clone the locked metatable to the new copy.
This would, however, come with a few downsides, including potential security and usability issues, as pointed out in the original RFC.
This approach is less flexible than the one proposed in this RFC as users couldn't define custom metatables for shallow copies and has no practical benefits that do not violate security guarantees.
