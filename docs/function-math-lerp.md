# math.lerp

## Summary

Add `lerp` (linear interpolation) to the standard `math` library, which takes in two numbers, as well as a fractional value that interpolates between the two.

## Motivation

`lerp` is a very common mathematical formula and should be natively supported in Luau for ease-of-use and faster execution. Despite potential floating-point imperfections (which we will discuss later), the Luau `math` library already contains many floating-point-based formulas and we can design `math.lerp` to work in effectively 100% of use-cases.

## Design

The base logic without edge-cases will look as follows, but implemented in C++. As per already-existing `:Lerp` logic in the Roblox level above Luau, this function will be **unbounded**, meaning that `alpha` does not have to strictly be a number between 0 and 1, allowing for linear extrapolation. This also means that numbers that approach `inf` or `-inf` should return `inf` or `-inf` respectively to avoid overflow.

```lua
function math.lerp(x: number, y: number, alpha: number): number
    if x == y then return x end
    if alpha == 0 then return x end
    if alpha == 1 then return y end
    
    return x + (y - x) * alpha
end
```

When implementing a `lerp` function, we can judge how accurate the function is by the following criteria (credit to [this](https://github.com/rust-lang/rust/issues/86269#issuecomment-869108301) post about the nature of `lerp`). If the `lerp` function can meet all of these criteria, then the function is effectively perfect.
- Is the function **exact**? This means that `math.lerp(x, y, 0) == x` and `math.lerp(x, y, 1) == ` exactly.
- Is the function **consistent**? This means that `math.lerp(x, x, alpha) == x`, despite the value of `alpha`.
- Is the function **monotonic**? This means that if `y` is strictly greater than `x`, `math.lerp(x, y, alpha)` should also strictly increase as `alpha` strictly increases, and vice versa for strictly decreasing. Now, due to the limitations of `double` precision, if `x` and `y` are very close together, there may be overlap where multiple values of `alpha` may correlate to the same result. This is ultimately unavoidable with any kind of `double` and `float` math, as we can observe the same behavior with `math.sin`, `math.cos`, etc.

## Drawbacks

As mentioned in the [Design](##design) section, the naïve implementation of `lerp` may introduce precision error, and all of the edge-cases will need to be accounted for.

## Alternatives

The only alternative solution would be to continue letting developers define their own lerp functions, placing it in a module and calling `require` all over the place, or placing it in a global scope such as `shared.lerp` or `_G.lerp` for easy access. However, this prohibits any Luau type-checking, and since `lerp` is an incredibly widely-used function, letting pure C++ handle the calculations can drastically improve performance.
