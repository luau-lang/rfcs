# vector.project

**Status**: Proposed

## Summary
Add the function `vector.project` to the vector library, which returns the vector projection of one vector onto another - the component of the first vector that is parallel to the second.

## Motivation
Projecting one vector onto another is a fundamental operation that comes up constantly in gameplay and simulation code: decomposing a velocity into components along and perpendicular to a surface (wall-sliding), constraining movement to an axis, camera-relative movement, and computing how far along a direction a point lies.

Today developers must spell this out by hand as `b * (vector.dot(a, b) / vector.dot(b, b))`. That expression is verbose and error-prone. The most common mistake is dividing by `vector.magnitude(b)` (one normalization) instead of the squared magnitude, or normalizing `b` first and forgetting the second factor. A first-class library function removes that and, like the rest of the vector library, opens the door to native-codegen and fast-call optimizations that a user-defined implementation cannot reach.

Both Unity (`Vector3.Project`) and Godot (`Vector3.project`) ship this operation, and Roblox is planning to expose an equivalent as a proposed `Vector3:Project` method on its own datatype.

## Design
The function `vector.project` will be added to the vector library that behaves similar to the following user-defined function:

```luau
function vector.project(a: vector, b: vector): vector
      return b * (vector.dot(a, b) / vector.dot(b, b))
end
```
`vector.project(a, b)` returns the component of a in the direction of b. The result always lies on the line spanned by b.

If b is the zero vector, the division is undefined and the result is a vector whose components are NaN. This is consistent with how vector.normalize and vector division already propagate NaN/Inf on degenerate input rather than silently clamping.

The implementation supports 4-wide vector mode, since it is expressed entirely in terms of vector.dot and scalar-vector multiplication.

Related operations developers can build on top of it:
- Scalar (signed) projection length: `vector.dot(a, vector.normalize(b))`.
- Rejection (perpendicular component): `a - vector.project(a, b)`.

## Drawbacks
This adds another library function, and most likely would require another fast-call slot for any meaningful performance gain in an interpreted runtime.

The zero-length behavior (returning NaN rather than the zero vector) differs from Unity, which clamps to zero. This is a deliberate choice for consistency with the rest of the Luau vector library, but it may surprise developers coming from other engines.

## Alternatives
Do nothing and leave people to write their own implementation. This forgoes the ergonomic, portability, and performance opportunities, and leaves the common squared-magnitude error in place.

Return the scalar projection (a number) rather than the projected vector. This is less general - the vector form is the established convention in Unity and Godot, and the scalar is trivially `vector.dot(a, vector.normalize(b))`.

Provide projection onto a plane (given a normal) instead of onto a vector. This is a different operation, expressible as the rejection `a - vector.project(a, normal)`, and is out of scope for this RFC.

Guard the zero-length case to return the zero vector (Unity's behavior). Rejected to stay consistent with existing Luau vector math, which propagates NaN/Inf rather than clamping.

## Prior Art
- Unity exposes `Vector3.Project(vector, onNormal)`, which returns the zero vector when onNormal has near-zero magnitude.
- Godot exposes `Vector3.project(b)` as an instance method with the same mathematical definition.
- Within Luau, this parallels vector.lerp (https://rfcs.luau.org/function-vector-lerp.html) and the existing `vector.dot`, `vector.cross`, and `vector.normalize` functions in both naming and semantics, and it mirrors Roblox's proposed `Vector3:Project` datatype method so that engine and standard-library APIs stay aligned.
