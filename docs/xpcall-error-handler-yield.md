# Feature name

## Summary

Allow `xpcall` error handlers to yield.

## Motivation

Lua 5.2+ `xpcall` can yield both the main function and error handlers. Luau can only yield the main function but not the error handler.

The Luau website says that `xpcall` yielding semantics have been ported from Lua 5.2. However, this isn't exactly correct, only the main function can yield, such an oversight can lead to potential confusion with users. This also adds a layer of incompatibility with vanilla Lua that's not needed.

The biggest issue is that the yield semantics are very un-intuitive, causing potential issues with novice users. Especially when the debug semantics aren't very user-friendly.

Furthermore, if one were to use a feature that would require yielding in the error handler, they would have to implement some kind of event wrapped functionality to be able to use the desired feature or use a different alternative to `xpcall` altogether.

## Design

Allow the second argument of `xpcall`, which is the error handler function to yield.

## Drawbacks

Potential performance implications and increased complexity in call semantic handling. Probably very minor though.

## Alternatives

One alternative to this is to use promises. Many users however prefer the vanilla `pcall` and `xpcall` functions for a variety of reasons.

Another alternative is to define a `yxpcall` function like so:
```lua
local function yxpcall(f, callback, ...)
	local args = table.pack(pcall(f, ...))

	if not args[1] then
		callback(args[2])
	else
		return table.unpack(args, 2)
	end
end
```
Having to define a specific `yxpcall` function just to allow a function to yield but having otherwise identical semantics feels cumbersome and frankly smells of the design stupidity of the deprecated `ypcall` function and most novice users may not be experienced enough to even pinpoint the source of the issue.
