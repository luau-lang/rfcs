# Do not change init.luau or relative require semantics

## Summary

There is no need to change the semantics of relative requires or init.luau behavior, so we should not do so.

## Motivation

There is currently a large amount of debate, and open RFCs, about changing require semantics for Luau due to a Roblox incompatibility issue with "script-folders", the idea of a script having children. This RFC aims to solidify that Luau should not conform to Roblox, because Luau's init.luau & relative path semantics are fine.

The current semantics are widely used, and similar to other languages with relative requires and an equivalent of `init.luau`. The current semantics also make sense, and they do not have any large issues.

## Design

Do not change the behavior of requires when it comes to init.luau & relative requires.

## Drawbacks

The mismatch between the type of `...` in function declaration (`number`) and type declaration (`...number`) is a bit awkward. This also gets more complicated when we introduce generic variadic packs.

## Alternatives

Don't implement this RFC, and instead rely on the Luau maintainers to make the call. This is kind-of fine, because regardless of what the outcome is, the vast majority of Luau users are not using require-by-string. However there are a lot of projects which do rely on the current semantics, and changing them to conform to Roblox when the ultimate issue is down to "how do we express requiring children in Roblox" isn't great.
