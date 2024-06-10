# Stable Sort

## Summary

Provide a stable sorting algorithm in the table library.

## Motivation

Luau's existing algorithm(s) for `table.sort` are unstable. (It first attempts a quicksort, and falls back to a heapsort when the quicksort takes too long. I expect this hybrid method has O(n log n) worst-case runtime, unlike raw quicksort which has O(n^2) worst-case.)

Sometimes one needs a stable sort instead. As a toy example, say your table contains objects of type `{key: number, value: number}`, where these should be compared/ordered only by their keys, and a table may contain multiple such objects with the same key. In some applications, you may you want objects with the same key to retain their relative order through a sort. For example, if you're using an array of these objects as an association list that can be temporarily modified by appending objects that reaassociate keys to new values, but want to be able later to discard/pop off the most recent assignments. Another application is if you want to sort by several different keys in sequence, but preserving the effects of the earlier sorts in the results of the final sort when elements are tied by the final sort key. See [this StackExchange entry](https://stackoverflow.com/questions/1517793/what-is-stability-in-sorting-algorithms-and-why-is-it-important) for more about the usefulness of stable sorts.

The [ways currently available](#Alternatives) to achieve a stable sort in Luau have signicant costs in time and memory. Also, though one doesn't always need a stable sort, the contexts where it is needed are many and various. If everyone has to implement a stable sort on their own, either from scratch or piggybacking on top of the unstable sort in the `table` library, their implementations will have less review and less thorough testing than a library version.

I propose to instead provide a stable sorting algorithm in Luau's `table` library. It could initially be behind a feature flag, or offered in parallel to the existing unstable sort. If its performance can be made comparable to the existing algorithms, one may consider having it be the only sort method provided.


## Design

The best stable sorting algorithms I know of are Python's [timsort](https://en.wikipedia.org/wiki/Timsort) and [block merge sort](https://en.wikipedia.org/wiki/Block_sort). Both are O(n log n) worst-case algorithms. I favor block-sort because it has variants requiring only constant space overhead (and this overhead can be kept very small, at some performance cost). The C implementations I've been testing are based on code from the [Grail Sort project](https://github.com/HolyGrailSortProject/Rewritten-Grailsort).

The proposed API would be:

`table.stable_sort <V>: (t: {V}, lt: ((V, V) -> boolean)?, reverse: boolean?) -> ()`

(Or alternatively, this could replace `table.sort` but be behind a feature flag.)

The optional third argument inverts the direction of the comparison being used (whether that's the default `lua_lessthan`, or a user-supplied comparison function). The infrastructure for this is a natural part of the sorting implementation anyway, so it seems a shame not to expose it in the user-facing API. This can avoid the overhead of an extra Lua function call for each comparison, when all one wants to do is invert the order of a search. (It'd also be straightforward to expose such an optional third argument on the existing unstable `table.sort` function.)

The block-sorting algorithm internally needs a table rotation algorithm, and we might as well also expose that as:

`table.rotate <V>: (t: {V}, leftwards: number) -> {V}`

This mutates the table in place but also returns it, which is sometimes useful. (Though one may argue it'd be more consistent with the table library's API to have this function return `()`.)

The Grail Project's block-sorting implementation I've been testing has three variants:

1. The simplest code uses no buffer space, and does all of its swapping and merging inside the original table. (With also a few temporary variables on the C stack, of course.) This is the least performant version, but the simpler code and low memory overhead may speak in its favor. Also it may be easier to ensure GC consistency that way. (A sort using the default comparison function shouldn't see any GC activity during its progress, but if we allow user-supplied comparison functions all bets about that are off.)

2. A second version uses a constant-sized buffer, for example an array of 512 table elements. The code I'm testing allocates this on the C stack, but it might instead be made a new field on a `lua_State`.

3. A third version allocates a single dynamic buffer for each sort, half the size of the table array being sorted. This is the most performant variant.

The code for versions 2 and 3 is nearly the same, differing only in the initial setup.

### Benchmarks

Here are some benchmarks sorting random key/value pairs. The "few key-collisions" tests had keys drawn randomly from a range ten times as large as the number of elements. The "many key-collisions" tests had keys drawn from range one-tenth as large as the number of elements. Results are the average of 1000 trials sorting different random data, divided by the time taken by the `qsort` from my machine's libc to sort copies of the same arrays. All the sorts use a custom comparison function.

Different systems implement `qsort` differently; it's not always quicksort and not even always unstable. But on my system (OS X 10.15) I can confirm it is a version of quicksort, using the technique of "median selection." These benchmarks are only giving a crude comparison of (my Grail-derived implementation of) block-sort versus one optimized quicksort algorithm (not against Luau's particular implementation of a hybrid quicksort/heapsort).

Lower numbers are better, so this table represents that block-sort is 74-95% slower than my system's qsort on arrays with 100 elements and few key-collisions, but that it can be somewhat faster than qsort on larger arrays of that kind of data (6-12% faster than qsort on arrays with 100k elements). WHen there are very many key-collisions (roughly one-tenth as many keys as elements), stably sorting will persistently be slower than qsorting, even on larger arrays (14-22% slower on arrays with at least 100k elements).


Number of elements        | Variant 1 | Variant 2 | Variant 3
--------------------------|-----------|-----------|----------
100,      few collisions  |   1.74    |   1.92    |   1.95
1000,     few collisions  |   1.17    |   1.14    |   1.18
10k,      few collisions  |   0.97    |   0.92    |   0.93
100k,     few collisions  |   0.94    |   0.88    |   0.88
1million, few collisions  |   0.98    |   0.95    |   0.91
100,      many collisions |   2.76    |   3.05    |   3.13
1000,     many collisions |   1.77    |   1.78    |   1.79
10k,      many collisions |   1.36    |   1.26    |   1.27
100k,     many collisions |   1.22    |   1.15    |   1.15
1million, many collisions |   1.21    |   1.16    |   1.14

Comparing block-sort to other stable sorting algorithms, [this site](https://github.com/BonzaiThePenguin/WikiSort) --- which uses a somewhat different implementation of block-sort --- says it's at worst 5% slower (and on some kinds of data very many times faster) than the `stable_sort` method in the C++ STL.


## Drawbacks

1. A C implementation of block-sort is inevitably going to be more complex than Luau's existing quicksort/heapsort hybrid. It'll also be more complex than a naive merge sort (which would also provide stability but at the cost of substantially more buffer space). Variant 1 described above (with no external buffer) is about 500 lines of code. The other versions add about another 200 more. For comparison, the existing sort algorithm in `ltablib.cpp` (excluding the entrypoint function exposed to Lua, and the swap and comparison functions), is about 100 lines of code.

2. If we implemented a stable sort algorithm alongside the existing unstable sorting algorithm(s), it would expand the API somewhat, also it'd make the Luau runtime/engine a bit bigger. If we ended up *replacing* the unstable sorting algorithm with a stable sort, there will be some circumstances in which the replacement is slower. But presumably this replacement wouldn't be entertained unless the performance impact seemed on balance to be acceptable.


## Alternatives

Currently-available ways to accomplish a stable sort would be:

1. Add an extra field to all of the objects in your array, which reflects the element's original position in the table, and provide a custom sort comparison function to order first by key then break ties using this extra field. If you're not able to add an extra field directly to the objects in your array, then wrap them in a further table that contains this extra field and the original object. With these extra bits of structure in the array being sorted, and the custom comparison function, you can use the existing unstable sort algorithm(s) in the `table` library, but preserve the relative order of objects with the same key. But creating and disposing the extra structure will add significant overhead in time and memory usage/GC work. In some contexts the additional work needed in the custom comparison function may also contribute to slowdown.

2. Write your own stable sort algorithm (such as a naive merge sort) in Lua. This will be substantially slower than the C-implemented sorting in `table.sort`, and it will also use a lot of additional memory. As mentioned in [the Motivation section](#Motivation) above,  if everyone who needs a stable sort has to implement their own, there will be more opportunities for error, since each implementation will have less review and less thorough testing than a library version.

3. Write your own stable sort algorithm in C, using the Lua/Luau API. This will be a lot faster than alternative 2, but it will still be slower than one might hope for, as the API for 3rd-party modules doesn't provide direct table access, and calls like `lua_rawseti` involve unnecessary overhead (checking whether the table is readonly on each call, undoing any GC progress on the table, Lua stack manipulation, and so on). The issue about many individual implementations seeing less review and testing also remains.

