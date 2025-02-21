# math.map

**Status**: Implemented

## Summary

Add a `map` function to the `math` library, which maps a number from one range to another.

## Motivation

Numerical mapping is a very common operation within game development. With only a brief search of the CoreScript source on the Roblox Client Tracker, there are about [three](https://github.com/MaximumADHD/Roblox-Client-Tracker/blob/9add9114d12e24cf08fa5680dbc95ace2af5c981/scripts/CoreScripts/Modules/PlayerList/Components/PresentationMobile/PlayerListApp.lua#L51) [different](https://github.com/MaximumADHD/Roblox-Client-Tracker/blob/9add9114d12e24cf08fa5680dbc95ace2af5c981/scripts/PlayerScripts/StarterPlayerScriptsCommon/RbxCharacterSounds.lua#L55) [implementations](https://github.com/MaximumADHD/Roblox-Client-Tracker/blob/9add9114d12e24cf08fa5680dbc95ace2af5c981/scripts/PlayerScripts/StarterPlayerScripts/PlayerModule.module/CameraModule/CameraUtils.lua#L53) of this function, each having about five to ten uses per context.

A native function would help boost ergonomics and provide potential performance benefits in these cases.

## Design

A new function, `math.map`, will be added and will act equivalently to the following user-defined function.

```luau
function math.map(x: number, inmin: number, inmax: number, outmin: number, outmax: number): number
	return outmin + (x - inmin) * (outmax - outmin) / (inmax - inmin)
end
```

Just like common Luau implementations, `x` is allowed to be outside the input range. Since clamped mapping is not as widely used and can be easily replicated using `math.clamp`, it would not be supported.

## Drawbacks

This adds yet another function to the standard library.

The function may also not provide enough performance improvements to warrant its addition, so implementations of this should be benchmarked.

## Alternatives

Not doing anything and letting people continue writing their own mapping functions. This would keep the same problems as before.

If the use of clamped mapping turns out to be more widespread, then adding an optional sixth argument to enable clamping could be considered. Not doing this may lead to the same problem of people writing their own implementations as before.
