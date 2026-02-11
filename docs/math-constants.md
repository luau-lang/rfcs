# Math constants

## Summary

Expand the constants provided in the `math` library.

## Motivation

With the introduction of [`math.isnan`](https://rfcs.luau.org/math-isnan-isfinite-isinf.html), Luau has the opportunity to add a new constant representing `NaN` (Not a Number).
As linting support for Luau continues becoming more robust, we can discourage patterns like `0/0` to generate `NaN` and instead provide the constant directly through the math library.

Without the `math.isnan` API, this could have created confusion in the past:
```luau
if maybeNan == math.nan do
    print("This line will never be hit: comparing to NaN always returns false.")
end
```

With the new API, however, this is now easier to use:
```luau
local maybeNan = math.nan
if math.isnan(maybeNan) do
    print("This line will be hit!")
end
```

While adding `NaN`, we also have an opportunity to modernize the set of constants available through the `math` library.
Some history:
- Lua 5.0 (2003) introduced `math.pi`, representing the mathematical constant $\pi$.
- Lua 5.1 (2006) introduced `math.huge`, mapping directly to infinity as defined by the [IEEE 754 standard](https://en.wikipedia.org/wiki/IEEE_754).
- Lua 5.3 (2015) introduced integers and the `math.maxinteger` and `math.mininteger` constants alongside them.

Luau currently only supports `math.pi` and `math.huge`.
Integer types are not supported in Luau, so `math.maxinteger` and `math.mininteger` wouldn't make much sense, but adding other commonly used constants would be an easy value-add for users of the language.

## Design

In addition to `math.pi` and `math.huge`, we propose adding the following constants.

- `math.nan`: some `NaN` value as defined by IEEE 754.
    - The exact representation (payload and sign) is unspecified.
    - Comparing to `math.nan` will always return `false`; lint rules recommend using `math.isnan`.
- `math.e`: the mathematical constant $e = 2.71828182845904523536$.
- `math.phi`: the golden ratio, defined as $\phi = 1.61803398874989484820$.
- `math.sqrt2`: the square root of `2`, defined as equal to `math.sqrt(2)`.
- `math.tau`: the mathematical constant $\tau$, defined as equal to `2 * math.pi`.
## Drawbacks

- The semantics of `math.nan` are unique and somewhat unexpected for users not already familiar with IEEE 754's floating-point arithmetic rules.
- Adding new constants increases the API surface of the `math` library and diverges further from Lua 5.1.

## Alternatives

- Do nothing: continue requiring users to define their own constants (e.g., `local nan = 0/0`, `local e = math.exp(1)`).
- Only add `math.nan` without the additional constants, deferring modernization for a future RFC.
- Rely on these constants being provided through external libraries.
