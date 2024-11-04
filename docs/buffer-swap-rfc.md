# buffer.swap(b: buffer, x: number, y: number, range: number?)

## Summary

`buffer.swap(b, x, y, range?)` swaps a variable `range(?=1)` of bytes stored at offsets `x` and `y` of buffer `b`.

## Motivation

The absence of a byte-range-swap function implies that for any variable range of bytes that need to be swapped, these bytes would have to be normalized as strings (see *Design*), *then* written back into the buffer. Although the implementation works, it would make more sense to have a buffer function that interally manipulates the desired offsets/range of bytes instead of the provided method where the byte ranges are interpreted as strings to achieve the same result.

In short, the current swap implementation has a string "layer" (unneccessary complexity/operations/conversions? (conjecture)), whereas the proposal for `buffer.swap( ... )` internally manipulates the byte ranges without any string interpretations etc.

## Design

An example of a `buffer.swap( ... )` implementation for any `range` range of bytes:

```lua
--- swaps range of bytes stored at offsets `x` and `y` \
--- `range (?=1)` range of bytes to swap
local function swap(b: buffer, x: number, y: number, range: number?)
	local z = buffer.readstring(b, x, range or 1)
	buffer.writestring(b, x, buffer.readstring(b, y, range or 1))
	buffer.writestring(b, y, z)
end

local b = buffer.create(8) --- create 8-width buffer
buffer.writef32(b, 0, math.pi) --- store 3.14... at 0
buffer.writef32(b, 4, 2*math.pi) --- store 6.28... at 4

swap(b, 0, 4, 4) --- buffer.swap(b, 0, 4, 4)

print(buffer.readf32(b, 0)) --- 6.28... is now at 0
print(buffer.readf32(b, 4)) --- 3.14... is now at 4
```

## Drawbacks

Real use-cases may be sparse, where this rfc could be interpreted as having added bloat to the buffer library.

## Alternatives

`is:pr is:open buffer.swap` search query on `/luau-lang/rfcs/pulls` returns no results.

`buffer.swap[u/i/f/string/...][8/16/32/64/...](b, x, y, range?)` (a case for each buffer type) was initially considered but that would very quickly become a nightmare for managing code. The provided working implementation is shown because (despite normalizing bytes as strings) it most-closely represents the goal of this rfc.
