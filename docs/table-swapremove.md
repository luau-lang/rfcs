# `table.swapremove`

## Summary

This RFC introduces a new `table` library function, `table.swapremove(t, i)`, that implements the swap-and-pop algorithm for an array.

## Motivation

Swap-and-pop is a very common algorithm used to remove an element from an array in O(1) by replacing said element with the final element, and then removing that last element. This is significantly faster, especially for larger arrays, than a regular `table.remove`-like operation, which must shift all subsequent elements to preserve a contiguous layout. It is used frequently in places like ECS implementations, particle systems, object pooling, and plenty more. A brief demonstration is shown below:

```luau
local myFavouriteFruits = {"apple", "banana", "kiwi", "orange"}
table.swapremove(myFavouriteFruits, 2)
print(myFavouriteFruits) -- {"apple", "orange", "kiwi"}
```

Currently, to perform swap and pop, one might usually do the following:
```luau
t[i], t[#t] = t[#t], nil
```
This obscures the intent of performing an unordered removal and leaves the details of the operation to individual implementations. For example, a manual implementation does not return the removed value like `table.remove`, does not consistently validate (namely, throwing an error for non-numeric indices) or floor (for non-integers) the provided index, and it is certainly less clear than a dedicated `table.swapremove`. It also has unexpected behaviour for negative indices or indices that are greater than maxn, namely turning it into a mixed table without any validation, and turning the entire array sparse respectively.

A dedicated `table.swapremove` function would provide a single, well-defined abstraction for this common operation, offering behavior that is consistent with `table.remove`. It would also complete the table library by providing an unordered counterpart to `table.remove`. 

As with table.insert and table.remove, users can already implement this operation themselves. The value of a standard library function is not enabling something impossible, but providing a clear, consistent, and optimized implementation.

Rust (`Vec`'s `swap_remove`), Zig (`ArrayList`'s `swapRemove`), and Unreal Engine (`TArray::RemoveAtSwap`) implement this, so it is not something entirely novel, though these are not as motivating given that all of these are statically typed languages, and Luau is not.
 
## Design

The proposed signature would be very similar to that of the existing `table.remove`:
```luau
table.swapremove<T>(t: {T}, pos: number): T?
```

`table.swapremove(t, pos)` will remove the element at the given position by moving the last element of the array into its position, and then return the element that was at the given position before the swap. It will set the last element to nil. Like `table.remove`, it does nothing on integer indices which do not belong to [1, maxn] (though errors on non-positive indices, like non-numeric indices), floors non-integer indices, throws errors for non-numeric indices, uses `pos = #t` if `pos` is omitted, and returns specifically the removed element (as opposed to the swapped element).

For arrays with holes, calling `table.swapremove` on an index with a nil value will not silently fail, and will as prescribed move the last element to this index and return nil correctly, as it is the removed element.

Calling `table.swapremove` beyond the maximum numerical index of the array, or on an index below 1, should do nothing. This may seem odd given we do not do this for holes in the array, but an alternative would be to continue as normal, which has undesirable effects. Consider the following:
```luau
local myFavouriteFruits = {"apple", "banana", "kiwi", "orange"}
table.swapremove(myFavouriteFruits, 99)
```
Swapping the last element with element 99 is already a major issue. Firstly, if we naively did `myFavouriteFruits[99] = myFavouriteFruits[4]`, this results in the table becoming `{[1]: "apple", [2]: "banana", [3]: "kiwi", [4]: "orange", [99]: "orange"}`. This introduces a large gap in what was previously a contiguous sequence, and turns it into a sparse array. Creating holes in this manner is undesirable and gives perhaps classically unexpected results, including #t (4) and table.remove (removes `[4]: "orange"`).

The second part, removing the last element, is also now at best ambiguous. It is unclear whether we remove the original last element, at index 4, or the new "last" element, at index 99. Using the former is what `table.remove` would do on this mutated array, but again that still leaves holes in the array which we did not have before. Using the latter would trivially have made the swapremove operation a no-op. There is no sensible way to swap-and-pop in this way.

Calling table.swapremove with a non-positive index should be an error. Consider the following:

local myFavouriteFruits = {"apple", "banana", "kiwi", "orange"}
myFavouriteFruits[-4] = myFavouriteFruits[#myFavouriteFruits]

This introduces a negative key into what was previously a contiguous array, turning it into a mixed table. This is an unexpected result for an operation whose purpose is simply to remove an element from an array. It is therefore safer and clearer to reject non-positive indices.

## Drawbacks

It is yet another function to the table library that has a relatively simple implementation, like `table.insert` or `table.remove`, that users could reasonably implement themselves as discussed earlier.

It is not immediately apparent to an unfamiliar user whether or not `swapremove` would return the swapped element or the removed element.

There is a clear inconsistency between how we handle different cases. We proceed normally for nil values in [1, maxn], but do nothing for nil values outside this.
We also try and preserve parity where possible with `table.remove` but we do throw errors for non-positive indices whereas `table.remove` does not.

## Alternatives

With regards to unacceptable indexes, we could more perform a no-op for non-positive indices, which does improve parity with table.remove. However, this allows for silent mistakes like off-by-one errors (e.g. trying to index with 0 instead of 1).

We could also return the swapped element alongside the removed element. This is not recommended, though, as it turns the signature into this: 
```luau
table.swapremove<T>(t: {T}, pos: number): (T?, T?)
```
It is not immediately clear which one is the swapped element, and which one is the removed element, without reading documentation. Furthermore, it is ambiguous at best what to do if pos is omitted / set to `#t`; we could equally return nil (as we did not need to swap with anything), or return the last element, which is an implementation detail.

We could simply not implement this function. Users could continue to use the swap assignment.
