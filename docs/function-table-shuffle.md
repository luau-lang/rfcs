# table.shuffle function

## Summary

Add a `table.shuffle()` function to the table standard library. 

## Motivation

I propose this addition because it matches a lot with the table library, specially how it works with array-like tables and i think it is a feature that is missing.

## Design

The function usage would be very straightforward

```luau
local tbl = {1, 2, 3, 4, 5, 6}
table.shuffle(tbl)  
```

Also, the function would accept an optional seed argument:

```luau
type table = typeof(table) & {
  shuffle: (tbl: { any }, seed: number?)
}
```

It could use a native Fisher-Yates implementation to shuffle the table around. I found out this could be useful when i was experimenting with permutation tables for my noise implementation.

```luau
local permutation = { 0, 1, 2, 3, 4, 5, 6, 7 ... 254, 255 }
function perlin(x: number, y: number, z: number, seed: number?): number
  -- Shuffles the permutation table with the provided seed or with 0 if no seed is set.
end
```

## Drawbacks

1. API size increase: This would make the API bigger and more complex;
2. Custom implementation: Users can already have implemented different shuffle algorithms in their programs.
3. Runtime implementation: Some runtimes could have added a function with the same name with a different algorithm.

## Alternatives

In Roblox, there is already a `:Shuffle()` method on the `Random` object.

## Prior Art

This addition would be very similar with other programming languages, like Python's `random.shuffle()` or Java's `java.util.Collections.shuffle()`.
