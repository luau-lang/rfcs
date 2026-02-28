# vector.sqrmagnitude

## Summary

Introduce a `vector.sqrmagnitude` function to the vector library. This function will return the squared magnitude of a vector.

## Motivation

The squared magnitude of a vector is commonly used in game development. It is used for distance checks, hitbox detection, and more. In all of these cases, the actual magnitude is unnecessary. Only the relative comparison matters. This makes the square root in `vector.magnitude` an avoidable cost. Currently, the idiomatic way to compute squared magnitude in Luau is to use `vector.dot(vec, vec)`. While this produces the correct result, there are problems with this:

First, ergonomics. The most common use case for squared magnitude is comparing the squared distance between two points, which looks like this:

```luau
local difference = vec1 - vec2
local range = 100

if vector.dot(difference, difference) < range^2 then
    --Within the range.
end
```

Developers are forced to store the difference in a local variable because `vector.dot` requires two arguments, or must subtract the two vectors twice for the arguments. With `vector.sqrmagnitude`, this becomes a single inline expression:

```luau
local range = 100
if vector.sqrmagnitude(vec - vec2) < range^2 then
    --Within the range.
end
```

Even outside of distance comparisons, it is much easier and more understandable to write `vector.sqrmagnitude(vec)` than `vector.dot(vec, vec)`.

Second, performance. When computing the squared distance between two points, the alternative `vector.dot(vec1 - vec2, vec1 - vec2)` computes the subtraction twice. A single parameter `vector.sqrmagnitude(vec1 - vec2)` avoids this redundancy. The subtraction is computed once and operated on directly. In case of `vector.dot`, even if someone stored the difference in a local, `vector.sqrmagnitude` would still potentially be faster since it is handled as a single intrinsic rather than going through the dot product way.

Third, ubiquity. Squared magnitude is widely used in code where only relative comparisons are required. Many math libraries and game engines expose it due to its frequency. 

## Design

Add one new function to the vector library:

`vector.sqrmagnitude(vec: vector): number`

Returns the squared magnitude of the given vector. This is equivalent to the sum of the squares of all of the vector's components, which includes the 4th component in 4-wide mode.

## Drawbacks

Just like all the other vector library functions, it is safe to assume this will use a fastcall slot. Adding this function will also increase the size of the standard library.

## Alternatives

Do not implement the function. People can continue using `vector.dot(vec, vec)` for the same result. 
