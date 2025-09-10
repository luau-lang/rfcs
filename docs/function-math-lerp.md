# math.lerp

**Status**: Implemented

## Summary

Add a `lerp` function to the `math` library, which performs linear interpolation between two numbers.

## Motivation

Linear interpolation is a very common operation in game development. It is present in standard libraries of some general purpose languages (C++, C#, Zig) and all shading languages (HLSL, GLSL, Slang). It can be used for implementing object motion, GUI sliders, color transitions, gameplay logic, etc.

Implementation of linear interpolation seems straightforward, but not all implementations have various numerical properties that can be important in practice. As such, a naive manual implementation may lead to unintended results.
Using `math.map` as a replacement with bounds 0..1 seems easy, but it results in worse performance and numerical properties compared to a specialized function, and is less intuitive than `lerp`. In presence of `math.map` and absence of `math.lerp`, users may gravitate towards the slower, numerically worse and less ergonomic solution.

A native function would help boost ergonomics and provide a good quality-performance tradeoff by default.

## Design

A new function, `math.lerp`, will be added and will act equivalently to the following user-defined function:

```luau
function math.lerp(a: number, b: number, t: number): number
	return if t == 1 then b else a + (b - a) * t
end
```

Just like common Luau implementations, `t` is allowed to be outside the input range. Since clamped mapping is not as widely used and can be easily replicated using `math.clamp`, it would not be supported out of the box.

Similarly, `a` and `b` range is allowed to be reversed; the behavior of suggested implementation is intuitive on a reversed range, and enforcing the additional restriction `a <= b` will reduce performance and limit use cases.

The next section will explain the rationale for choosing the proposed implementation.

## Rationale

In addition to performance, interpolation functions can be evaluated using a set of criteria:

- exactness: `lerp(0) = a`, `lerp(1) = b`
- monotonicity: `lerp(t1) >= lerp(t2)` for `t1 >= t2`
- determinacy: `lerp(a, b, t)` is NaN only if `t` is infinite or `a`/`b`/`t` are NaN
- boundedness: `lerp(t)` is in `[a, b]` interval for any `t` in `[0, 1]`
- consistency: `lerp(x, x, t) == x` for all `t`

Some applications may require some of these properties but not others. Enforcing all of these properties is expensive, with the most difficult property being determinacy, as many pairs of `b` and `a` of opposite signs will overflow the `b-a` computation. [C++ std::lerp](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0811r3.html) uses a different formula to handle interpolation for `a` and `b` of different signs.

In practice, exactness can be crucial for some applications as in common use cases, such as interpolating an object's motion from A to B, not reaching the exact final position may result in not triggering conditions based on comparing the result. Boundedness may be important when the result of `lerp` is an input to another function; for example, reverse trigonometric functions like `math.acos` will produce NaN for out of range inputs. Consistency can be valuable to avoid sub-pixel jitter in cases where the object is supposed to stay in place during an animation sequence where the position is being interpolated between two equal points. Similarly, monotonicity can be valuable to avoid sub-pixel jitter in reverse direction of motion in case the object is interpolated between two points.

Thus, ideally we would like to find a function that is exact, monotonic, bounded and consistent; and, of course, enforcing these properties should come at the minimum performance cost.

The following commonly encountered formulas have been evaluated for numerical properties, using a combination of numerical reasoning, exhaustive searches and randomized fuzzing. The analysis has been performed for 32-bit floats; it should hold for 64-bit floats as well. Monotonicity is only checked in the `[0, 1]` interval; we also assume `a <= b` for the sake of this analysis.

```c++
// exactness 0, monotonicity 1, determinacy 0, boundedness 0, consistency 1
return a + (b - a) * t;

// exactness 1, monotonicity 0, determinacy 1, boundedness 0, consistency 0
return a * (1 - t) + b * t;

// exactness 1, monotonicity 1, determinacy 0, boundedness 1, consistency 1
return t == 1 ? b : a + (b - a) * t;

// exactness 1, monotonicity 1, determinacy 0, boundedness 1, consistency 1
return t < 0.5f ? a + (b - a) * t : b + (b - a) * (t - 1);

// exactness 1, monotonicity 0, determinacy 1, boundedness 0, consistency 1
return a == b ? a : a * (1 - t) + b * t;
```

The only two functions that seemingly satisfy our 4 (of 5) desired criteria are the implementation proposed in the design section, and another version that switches behavior around `t=0.5`. Both do not guarantee determinacy due to potential overflow in `b - a`, but seem to guarantee (empirically or provably) other properties for `a <= b` and `0 <= t <= 1`.

While switching direction around `t=0.5` can increase precision in certain cases, it is also noticeably slower: a branchless implementation requires 6/9 instructions on A64/X64 (absent `fma`) for the proposed implementation, and 9/16 instructions for the last variant of the lerp. The standard implementation should strike the best balance between correctness and performance, and the `t == 1` variant seems to fit the bill.

Our proposed implementation is trivially exact and consistent due to floating-point properties. Due to floating-point properties, it is monotonic with one possible exception of values around 1. Specifically, it's unclear what the relation is between:

- `a + (b - a) * (1-ulp)`
- `b`
- `a + (b - a) * (1+ulp)`

Due to monotonicity property, `a + (b - a) * (1-ulp) <= a + (b - a)`; due to floating point arithmetical properties, `a + (b - a)` should be at most within 1ulp from `b` if `b >= a >= 0`. Empirically, there are cases where it's `b-ulp`, `b` and `b+ulp`, depending on values of `a` and `b`. While it *seems* that our adjustment of the output for `t == 1` can't break monotonicity as it seems impossible that `a + (b - a) * (1-ulp)` is greater than `b`, or that `a + (b - a) * (1+ulp)` is less than `b`, this property has not been formally proven. As such, there's a risk that our implementation is not fully monotonic. C++ `std::lerp` includes a statement "monotonic except near t=1", and, for `a < b`, clamps the result of the lerp to ensure it's correctly ordered with respect to `b`.

Since the fuzzer has not been able to find a counter-example for our implementation, we assume that either this property does hold or, if it doesn't, the counter-examples are extremely rare. It is likely that the claim C++ makes requires non-default rounding direction; the evaluation has been performed with round-to-nearest, which is the default on all platforms Luau targets. Additionally, the tests from Microsoft STL implementation have been validated against this implementation (the only test on finite numbers that fails compares the sign of the returned zero, which our implementation does not make promises about). The final analysis also involved running the fuzzer with both `float` and `double` inputs for 10 hours using 10 cores each and found no counterexamples where monotonicity is broken, evaluating ~300B triples for each type. Reverse ranges (`a > b`) were also lightly tested for monotonicity in the same 10 hour period on 1 core and found no issues either over ~75B triples.

In the event monotonicity is indeed somehow broken around 1, the author still considers the `lerp` function described above practically useful and a good balance between performance and correctness.

> Note: the evaluation and analysis has been performed without fma; while it *probably* holds when `+` and `*` are fused, that has not been established.

Additional references:

- [C++ proposal for std::lerp](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p0811r3.html)
- [Rust discussion of fNN::lerp](https://github.com/rust-lang/rust/issues/86269)
- [StackOverflow discussion of lerp implementations](https://math.stackexchange.com/questions/907327/accurate-floating-point-linear-interpolation)
- [lerpfuzz.cpp used for analysis + tests](https://gist.github.com/zeux/ea70d4574a50cc819e5f02b6d7ebb1f2)

## Drawbacks

This adds yet another function to the standard library.

The implementation does not guarantee all possible properties of a `lerp` function, and represents a balance between performance and numerical quality; in specific applications, different properties may be required to reach a different set of tradeoffs.

## Alternatives

Not doing anything and letting people continue writing their own lerping functions.

Implementing the "perfect" `lerp` function from C++ standard (significantly slower, but guarantees determinacy), or the "lazy" `lerp` function from shading languages (no guarantees, maximum performance).

Choosing an idiosyncratic name (`mix`) or argument order (`t, a, b`).
