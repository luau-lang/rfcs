
# vector.lerp

**Status**: Implemented

## Summary
Add the function `vector.lerp` to the vector library, which performs linear interpolation between two vectors and is a direct parallel to [`math.lerp`](https://rfcs.luau.org/function-math-lerp.html).

## Motivation
Linear interpolation is just as common of an operation with vector math as it is with scalars. For example, Roblox provides a [`Vector3:Lerp(goal, alpha)`](https://create.roblox.com/docs/reference/engine/datatypes/Vector3#Lerp) method for their vector implementation.

Having a built-in library function for linearly interpolating vectors would help facilitate portability, ergonomic and performance improvements to vector math just like `math.lerp`.

## Design
The function `vector.lerp` will be added to the vector library that behaves similar to the following user-defined function:

```luau
function vector.lerp(a: vector, b: vector, t: number): vector
	return if t == 1 then b else a + (b - a) * t
end
```

This implementation mirrors the one for `math.lerp` and inherits the same mathematical properties of exactness, monotonicity, boundedness and consistency. It also supports 4-wide vector mode.

## Drawbacks
This adds another library function, and most likely would require another fast-call slot for any meaningful performance gain in an interpreted runtime.

The implementation of `Vector3:Lerp(goal, alpha)` in Roblox does not have the same guarantees of its numerical properties, i.e. it is not exact or deterministic. This may cause confusion because the two won't behave or return the exact same results for certain cases.

## Alternatives
Do nothing and leave people to write their own implementations. This forgoes any portability, ergonomic and performance opportunities, especially in cases where a numerically correct implementation is preferred and the context cannot take advantage of native codegen.

Instead of a scalar `t` argument, the function could be fully vectorized and take in a vector for the alpha. While theoretically useful, most use-cases only require a scalar alpha, and the concerns for performance and numerical correctness of a fully vectorized implementation outweigh its usefulness.

A different set of correctness tradeoffs could be chosen such as guaranteeing determinacy. This would come at the cost of performance and make `vector.lerp` inconsistent with `math.lerp`.
