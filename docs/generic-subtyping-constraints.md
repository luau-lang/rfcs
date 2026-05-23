# Generic Subtyping Constraints

## Summary

This RFC introduces generic constraints. This allows:

- Preserving type correlations between function inputs and outputs,
- Restricting generics to specific structures,
- Adding safety rails to functions that manipulate the inputs.

### Main Example

```luau
function merge<T: number | vector>(a: T, b: T): {T}
    return {a, b}
end
```

## Motivation

Luau currently lacks a way to express correlated subtypes between parameters without verbosity.

When using unions, type information is lost. Assume we want to create a function that takes in the same type for both `a` and `b`.

```luau
function merge(a: number | vector, b: number | vector): {number | vector}
    return {a, b}
end

merge(1, 1) -- ok!
merge(vector.zero, vector.one) -- ok!
merge(1, vector.zero) -- allowed in type checking, but not intended.
```

Even with generics, we are unable to constrain which types of variables are able to be passed into `merge`.

```luau
function merge<T>(a: T, b: T): {T}
    return {a, b}
end

merge("hello", "there!") -- allowed, but again not our intended behaviour.
```

But with generic constraints, `T` can only be instantiated with types assignable to `number | vector`.

```luau
function merge<T: number | vector>(a: T, b: T): {T}
    return {a, b}
end
```

### Preserving Subtypes

Roblox Instances are especially prone to losing subtype information.

```luau
local meshPart1: MeshPart, meshPart2: MeshPart

function transformParts(...: BasePart): {BasePart}
    local parts = {...}
    for _, part in parts do
        part.CFrame *= CFrame.new(1, 1, 1)
    end
    return parts
end

local parts = transformParts(meshPart1, meshPart2) -- returns '{BasePart}'
```

This is a really common footgun. Whenever a more refined type gets passed in, the array becomes unrefined. Changing this into a generic constraint would look like:

```luau
function transformParts<T: BasePart>(...: T): {T}
    ...
end

local parts = transformParts(meshPart1, meshPart2) -- stays as '{MeshPart}'
```

## Design

A generic parameter can be constrained by adding `:` after its declaration.

```luau
function f<T: Constraint>(x: T): T
```

This would constrain `T` to be a subtype of `Constraint`.

```luau
function merge<T: number | vector>(a: T, b: T): {T}
    return {a, b}
end

merge(1, 2) -- ok, T is 'number'
merge(vector.zero, vector.one) -- ok T is 'vector'
merge(1, vector.zero) -- not ok, no single T can satisfy both args
```

Meaning:

- `T: Constraint` implies that `T` must be assignable to `Constraint`.
- All occurrences of `T` in the function must resolve to the same inferred type. Types will not widen the given types to satisfy `Constraint`. This means that inference will never choose union types in order to resolve `Constraint`.
- If no `T` can satisfy `Constraint`, then the inputs are invalid.

Therefore,

```luau
function transform<T: BasePart>(x: T, y: T): T

local part: Part
local meshPart: MeshPart
local basePart: BasePart

transform(part, meshPart) -- not ok. generic constraints will not infer BasePart (the common supertype), nor will it infer 'Part | MeshPart'
transform(basePart, meshPart) -- not ok for the same reason as above
```

```luau
function printAdd<T: number | vector>(x: T, y: T)
	print(x + y)
end

local a = if math.random() > 0.5 then 1 else vector.zero -- number | vector
local b = if math.random() > 0.5 then 1 else vector.zero -- number | vector
printAdd(a, b) -- typing is ok. a, b are 'number | vector'.
			   -- but, this can cause a runtime error if type(a) ~= type(b)
```
Since the types passed into `printAdd` are `number | vector`, they already satisfy `T`.

Explicitly putting a type on a generic is allowed as well as long as `T` is assignable to `Constraint`. This follows the same rules and design as above.

```luau
printAdd<<number>>(1, 2) -- ok! T is explicitly 'number'
printAdd<<vector>>(vector.one, vector.zero) -- ok! T is explicitly 'vector'
printAdd<<number>>(vector.one, vector.zero) -- not ok. T is explicitly 'number', and 'vector' cannot be casted to 'number'
printAdd<<string>>("hello", "foo") -- not ok. 'string' is not assignable to 'number | vector'
```

### Type Aliases

Constraints apply to type aliases as well.

```luau
type getProperty<T, K: keyof<T>> = (dict: T, key: K) -> index<T, K>
type getProperty<K: keyof<T>, T> = (dict: T, key: K) -> index<T, K> -- also valid! lexical ordering does not matter because generic parameters are unordered
```

### Mutually Dependent Constraints & Recursive Constraints

```luau
type bad<T: {[K]: any}, K: keyof<T>> = ...
-- T refers to K, and K refers to T, not allowed
```

This is deliberate. Solving mutually dependent constraints greatly complicates generic inference. [F-bounded Polymorphism](https://en.wikipedia.org/wiki/Bounded_quantification#F-bounded_quantification) will also be out of scope for this RFC, but can be amended in the future if higher motivation is present.

## Drawbacks

- Adding generic constraints makes it really easy to over-generalize code.
```luau
function getName<T: { name: string }>( object: T ): string
    return object.name
end
-- why use a generic?
function getName( object: { name: string } ): string
    return object.name
end
-- both of these functions have exactly the same semantics
-- the generics add no meaningful value; it just makes the code more verbose and does unnecessary generic inference
```
- Unions are able to create a false sense of safety and can lead to runtime errors if not careful.
- Adding generic constraints is a massive step up in complexity for the type checker.

## Alternatives

- Use overloads. Overloads can perform a lot of what generic constraints aim to achieve, but is not scalable, and does not preserve subtype relationships. Overloads are also unable to represent relationships between generic parameters.
```luau
local merge: (
	(number, number) -> number &
	(vector, vector) -> vector 
) = function(a, b)
	return {a, b}
end
-- works, but is more verbose and if more types need to be added here (like integers), we would need to just keep on adding overloads.
```
- Do nothing. Workarounds exist, but lead to code duplication or verbosity. This includes having an extra typecast every time a function is used, writing duplicate functions or writing a custom type functions. 