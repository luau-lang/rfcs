# Feature name

## Summary

Allow `table.clone` to copy tables with locked metatables.

## Motivation

As proposed by the [`table.clone` RFC](function-table-clone.md), the function `table.clone` cannot create shallow copies of a table if it has a locked metatable.
Due to this limitation, it is very un-ergonomic to create shallow copies of arbitrary tables.
The current behavior of using `table.clone` on a table with a locked metatable is a hard-fail approach where `table.clone` spits out an error instead of having a soft-fail approach wherein a clone is generated without a metatable.
This hard-fail approach severely hinders the usefulness of `table.clone` due to in practice users having to re-implement the function in many scenarios.
Example use cases wherein the proposed behavior would allow for elegant behavior and where the current behavior would be extremely unergonomic are:

An example is a JSON serialization function that wants to strip metatables completely before attempting to serialize said table to JSON.
Other examples where users may want to strip metatables may be security layers that don't want unwanted behavior caused by metatables.
```lua
local function JSONEncode(tbl)
	return JSONSerialize(setmetatable(table.clone(newTbl), nil))
end

-- This wouldn't be possible nowhere near as elegantly with the current behavior and users would effectively have to re-implement table.clone in Lua.
-- The point of table.clone after all was to require users to now create shallow copy functions in Lua themselves.
```

Another example is where you do want to mirror the behavior of the original table after cloning as much as possible.
With the new behavior, the following code would be possible.
```lua
local new = {}

for k, v in old do
	new[k] = (type(v) == "table" and getmetatable(v) ~= nil) and setmetatable(table.clone(v), {
		__index = function(_, k)
			return old[k]
		end,
		__newindex = function(_, k, v)
			old[k] = v
		end,
	}) or v
end

return new
```
With the old behavior said code wouldn't be possible, and you once again would have to re-implement the shallow copy function manually. Even using `pcall` for `table.clone` wouldn't work here as the raw keys wouldn't be copied over.

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

The original RFC puts forth a rationale for restricting cloning tables with locked metatables due to security reasons.
This is a valid concern. The issue at hand being an avenue to assign metatables to unintended tables.
However, this is not an issue as the changes proposed in this RFC do not allow for such behavior.

Another potential drawback that could be levied at this proposal is a reduced debuggability of not hard-failing when the expected behavior would be to create a shallow copy with the metatable of the original table.
However, in practice the benefits of this RFC outweigh the drawbacks regarding debuggability, as most users are very likely not even aware, or desire the fact that `table.clone` can assign the metatables of the original table to the clone.

## Alternatives

An alternate mode of action would be to also clone the locked metatable to the new copy.
This would, however, come with a few downsides, including potential security and usability issues, as pointed out in the original RFC.
This approach is also less flexible than the one proposed in this RFC as users couldn't create shallow copies without metatables at all and has no practical benefits that do not violate security guarantees.
