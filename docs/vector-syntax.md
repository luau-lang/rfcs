# Vector constructor syntax

## Summary

Implement new syntax to construct vectors, `<x, y, z>`.

## Motivation

Currently, vectors are constructed using constructor functions. While this has worked, it can harm the readability of code by making it harder to distinguish between a vector and a function call, and can add lots of nested function calls to code.

## Design

Vectors can now be constructed using the syntax `<x, y, z?>`, with an additional `w?` parameter being added when 4-wide mode is enabled. All parameters must be of type `number`, and the first two parameters are required.

```lua
local newvector = <1, 2, 3>
vector.magnitude(<1, 2, 3>)

-- when using 2 components of a vector
local newvector2 = <1, 2>
vector.magnitude(<1, 2>)

-- in 4 wide mode
local newvector4 = <1, 2, 3, 4>
vector.magnitude(<1, 2, 3, 4>)

-- example of something that would error
<1, 2, 3> -- Expected identifier when parsing expression, got '<1, 2, 3>'
```

If a number is placed next to angle brackets used to construct a vector, the angle bracket will be interpertred as a comparison operator instead of being part of vector construction.

```lua
function Foo(...)
  for i, v in {...} do
    print(i, v)
  end
end

Foo(1 <2, 3, 4> 5)
-- 1, true
-- 2, 3
-- 3, false

Foo(1 <2, 3, 4>)
-- Expected identifier when parsing expression, got ')'
```

Vectors must be wrapped in parentheses when being used as the only parameter in a function call to prevent confusion with type parameters and generics.

```lua
function Bar(v: vector)
  -- code
end

Bar(<1, 2, 3>) -- ok!
Bar<1, 2, 3> -- not ok, will error.
```

This RFC does not propose any changes when printing or stringifying vectors. Printing vectors varies based on what runtime you use, and stringifying vectors will still return `"x, y, z"`.

## Drawbacks

This will add more syntax to Luau, making the language more complex and more bloated.

Angle brackets are already used for comparison, type parameters, and generics. Making them used for vector construction syntax would give them another use case and could make them more complicated to work with.

## Alternatives

Do nothing; vectors can already be constructed using `vector.create` or any other runtime provided vector constructor.

Use different syntax for constructing vectors instead of using angle brackets. However, other proposals would cause issues with other features in Luau or would be confusing to work with.
- `(x, y, z)` is often used for tuples and functions, so it would not work.
- `[x, y, z]` is used for table indexing, so writing out `newtable[[x, y, z]]` may confuse readers. People may also write `newtable[x, y, z]` expecting for it to index the vector.
- `vector(x, y, z)` would give the `vector` library a `__call` metamethod, which requires more discussion over whether this is a good idea.
- `|x, y, z|` isn't valid vector notation.