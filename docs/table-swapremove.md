# `table.swapremove`

## Summary

This RFC introduces a new `table` library function, `table.swapremove(t, i)`, that implements the swap-and-pop algorithm for an array.

## Motivation

Swap-and-pop is a very common algorithm used to remove an element from an array in O(1) by replacing said element with the final element, and then removing that last element. This is significantly faster, especially for larger arrays, than a regular `table.remove`-like operation, which must shift all subsequent elements to preserve a contiguous layout. It is used frequently in places like ECS implementations, particle systems, object pooling, and plenty more. A brief demonstration is shown below:

```lua
local myFavouriteFruits = {"apple", "banana", "kiwi", "orange"}
table.swapremove(myFavouriteFruits, 2)
print(myFavouriteFruits) -- {"apple", "orange", "kiwi"}
```

Currently, to perform swap and pop, one might usually do the following:
```lua
t[i], t[#t] = t[#t], nil
```
While this idiom is concise, it obscures the intent of performing an unordered removal and leaves the details of the operation to individual implementations. For example, a manual implementation does not return the removed value like `table.remove`, does not consistently validate (namely, throwing an error for non-numeric indices) or floor (for non-integers) the provided index, and may not update the VM's cached array length in the same way as the `table` library. A native implementation could improve performance.

A dedicated `table.swapremove` function would provide a single, well-defined abstraction for this common operation, offering behavior that is consistent with `table.remove`. It would also complete the table library by providing an unordered counterpart to `table.remove`. Users may implement swap-and-pop themselves, just as they might implement insertion with `t[i] = value` or append with `t[#t + 1]` = value. The value of `table.insert`, however, is not that it enables something previously impossible or impractical, but that it provides a clear, consistent, and optimised implementation of a common operation. This RFC proposes doing the same for unordered removal.

Rust (`Vec`'s `swap_remove`), Zig (`ArrayList`'s `swapRemove`), and Unreal Engine (`TArray::RemoveAtSwap`) implement this.
 
## Design

The proposed signature would be very similar to that of the existing `table.remove` (using the style of the [current documentation](https://create.roblox.com/docs/reference/engine/libraries/table#remove)):
```lua
table.swapremove(t: {any}, pos: number): Variant
```

`table.swapremove` behaves similarly to `table.remove`. Namely:
- Updates the VM's cached value for #t
- Does nothing (silently) on integer indices which have a `nil` value, floors non-integer indices, and throws errors for non-numeric indices
- Uses `pos = #t` if `pos` is omitted
- Returns the removed element

We propose using Rust and Zig's `swapremove` nomenclature (in luaucase), as opposed to Unreal Engine's `RemoveAtSwap`, or something else like `swapandpop` or `moveandpop`.

## Drawbacks

It is yet another function to the table library that has a relatively simple implementation, like `table.insert` or `table.remove`, that users could reasonably implement themselves as discussed earlier.
It is not immediately apparent to an unfamiliar user whether or not `swapremove` would return the swapped element or the removed element.

## Alternatives

We could simply not implement this function. Users could continue to use the swap assignment.
