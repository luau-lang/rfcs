# vector.lerp

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

The implementation of `Vector3:Lerp` in Roblox holds all of the numerical properties *including* determinacy (returns NaN if `t` is infinity or `a`/`b` are NaN), while the proposed function and `math.lerp` don't. This could cause confusion as the two don't behave exactly the same.

## Alternatives
Do nothing and leave people to write their own implementations. This forgoes any portability, ergonomic and performance opportunities, especially in cases where a numerically correct implementation is preferred and the context cannot take advantage of native codegen.

Instead of a scalar `t` argument, the function could be fully vectorized and take in a vector for the alpha. While theoretically useful, most use-cases only require a scalar alpha, and the concerns for performance and numerical correctness of a fully vectorized implementation outweigh its usefulness.

To match Roblox's `Vector3:Lerp` correctness guarantees, the implementation could also account for determinacy. This comes at the cost of performance and would make `vector.lerp` inconsistent with `math.lerp`.
