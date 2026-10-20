# math.sum

## Summary

``` math.sum(...: number): number ```

Add a ```sum``` function to the ```math``` library, which adds the numbers together.

## Motivation

Numerical addition is one of the most fundamental mathematical operations. 
Summing values is common in game development for damage calculations, currency totals, and statistical computations.

A variadic function is particularly useful when the number of values to sum is not known in advance.

A native implementation would improve ergonomics and could also enable implementation-specific optimizations.
## Design

Requesting a new function, ```math.sum```:

```lua

function math.sum(...: number) : number
  local total = 0

  for _, num in {...} do
    total += num
  end

  return total
end
```

## Drawbacks

This introduces another function to the standard library.

Additionally, any performance improvements may not be significant enough to justify the API expansion.

## Alternatives

Continue relying on user-defined helper functions.

While this is straightforward, it results in repeated implementations across projects and misses the opportunity to provide a standard, 
discoverable API for a common operation.
