# Precision parameter in math.round

## Summary
This RFC proposes introducing a second parameter to `math.round` to allow users to specify a number of decimals to round to.
```luau
print( math.round(3.1415, 2) ) -- Output: 3.14
```
In doing so, we:
- Improve user convenience by providing an implementation of the behavior for them
- Provide a marginally more performant alternative to existing implementations

## Motivation
The motivation for this proposal lies mostly in improving user convenience. Albeit very simple, the concept's current implementation (demonstrated in **Design**) has two major flaws:
1. It's a 'one-more-than-necessary' utility users have to carry around
2. Though the difference is negligible in most scenarios, it's computationally slower, even in `native`

The change, as proposed below, feels like a very simple and appropriate expansion of the standard library, and is inherently a performance microoptimization [thanks to the fastcall op (assuming `math.round` remains a fastcalled function)].

## Design
As of writing this, a well-rounded implementation of rounding to a specific number of decimals is something like so:
```luau
local function round_to( n: number, decimals: number? )
    assert(type(n) == 'number', 'first arg must be a number')
    assert((decimals == nil) or (type(decimals) == 'number'), 'second arg must be a number')
    decimals = math.max(decimals or 0, 0)
    local scale = 10 ^ decimals
    return math.round( n * scale )/scale
end
print( round_to(3.1415, 2) ) -- Output: 3.14
```
The proposed change is simple: introduce a second parameter to `math.round` which accepts an optional `number`. This second parameter, which might be called something like `decimals`, determines to what number of decimals to round a number (the `n` parameter) to.
```luau
print( math.round(3.1415, 2) ) -- Output: 3.14
```

This change would provide the same level of convenience as the above `round_to` function without impacting the, for lack of a better term, 'visual' size of projects (like carrying around a `round_to`-esque function does). And to some users, this may even feel intuitive (see [deviaze's comment on the PR for this RFC](https://github.com/luau-lang/rfcs/pull/108?notification_referrer_id=NT_kwDOC10tpbUxNTE5NTEwNDgzMzoxOTA2NTU5MDk#issuecomment-2708646405), which I will also admit I once tried).

This RFC won't make any suggestions for handling edge cases (ie. if the `decimal_places` input is a negative number, or a non-nil but invalid type) because I don't want to assume that I know the best way to handle them in the context of being a global API.

## Drawbacks
Implementing this change would increase the physical size of Luau, which is antithetical to Luau's 'minimal size principle'.

It may also be worth considering that because `math.round` has gone unchanged for so long, some users may be confused when seeing the new syntax.

## Alternatives
### A whole new function
One alternative could be to introduce a new function in the `math` library specifically for rounding to decimal places.
```luau
-- imagine 'todec' means 'to decimal [precision...?]'
print( math.todec(3.1415, 2) ) -- Output: 3.14
```
The problem with this approach is that it's a rather unnecessary cluttering of the `math` library, especially in comparison to the proposed, much simpler solution. While they both achieve the same thing, this approach feels like it 'takes up too much space'.

### Do nothing
Another alternative could be to continue doing nothing.

Truthfully, the idea of doing nothing doesn't resonate with me in any way, for two reasons:
1. I can't imagine there's truly anything to lose by making this change. This RFC acknowledges the impact the change will have on the size of Luau as a whole, but I still think that impact is negligible.
2. If perhaps the reason for doing nothing is that the usecase[s] are too infrequent/niche, to that I argue: I like to liken this RFC to a theoretical RFC for introducing `math.round` [in an alternate universe where it doesn't already exist]; said RFC would likely make a very similar case to this RFC, in that the existing implementation is simple (pick `math.ceil` or `math.floor` based on some context) but more or less unnecessary for users to carry around. If we imagine that, in this universe, Lua took the liberty of proposing & implementing this RFC for us (Luau), then, as part of us being a fork of Lua, it should [logically] be in our best interest to make such a change (assuming Luau upholds API design principles similar to Lua, which [so far] seems to be the case and [to my knowledge] has not explicitly stated being in opposition to/of).
