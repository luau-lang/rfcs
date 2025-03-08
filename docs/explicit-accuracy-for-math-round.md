# Explicit accuracy for math.round

## Summary
This RFC proposes introducing a second parameter to `math.round` to allow users to declare a degree of accuracy to round numbers to.
```luau
print( math.round(3.1415, 2) ) -- Output: 3.14
```

## Motivation
The motivation for this proposal lies solely in improving user convenience. Albeit very simple, the concept's current implementation (demonstrated in **Design**) has two major flaws:
1. It's a 'one-more-than-necessary' utility users have to carry around
2. Though the difference is negligible in most scenarios, it's computationally slower, even in `native`

The change, as proposed below, feels like a very simple and appropriate expansion of the standard library, and is inherently a performance microoptimization [thanks to the fastcall op].

## Design
As of writing this, a well-rounded implementation of rounding to a decimal place is something like so:
```luau
local function round_to( n: number, decimal_places: number? )
    assert(type(n) == 'number', 'first arg must be a number')
    assert((decimal_places == nil) or (type(decimal_places) == 'number'), 'second arg must be a number')
    decimal_places = math.max(decimal_places or 0, 0)
    local scale = 10 ^ decimal_places
    return math.round( n * scale )/scale
end
print( round_to(3.1415, 2) ) -- Output: 3.14
```
The proposed change is simple: introduce a second parameter to `math.round` which accepts a `number?`. This second parameter, which might be called something like `decimal_places`, determines to what number of decimal places (degree of accuracy) to round a number (the `n` parameter) to. This change would provide the same level of convenience as the above `round_to` function without impacting the, for lack of a better term, 'visual' size of projects (like carrying around a `round_to`-esque function does).
```luau
print( math.round(3.1415, 2) ) -- Output: 3.14
```
`math.round`'s new type signature would be `( n: number, decimal_places: number? ) -> number`.

This RFC won't make any suggestions for handling edge cases (ie. if the `decimal_places` input is a negative number, or a non-nil but invalid type) because I don't want to assume any specific way is the best way.

## Drawbacks
Implementing this change would increase the physical size of Luau, which is antithetical to Luau's 'minimal size principle'.

It may also be worth considering that because `math.round` has gone unchanged for so long, some users may be confused when seeing the new syntax.

## Alternatives
One alternative could be to introduce a new function in the `math` library specifically for rounding to decimal places.
```luau
-- imagine 'todec' means 'to decimal [precision...?]'
print( math.todec(3.1415, 2) ) -- Output: 3.14
```
The problem with this approach is that it's a rather unnecessary cluttering of the `math` library, especially in comparison to the proposed, much simpler solution. While they both achieve the same thing, this approach feels like it 'takes up too much space'.

Another alternative could be to continue doing nothing. My only rebuttal is that I think there's nothing to lose by making this change, however, I recognize that that's largely my personal bias, and that it may also be, in some ways and to some extent, an uneducated take.
