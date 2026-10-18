# Syntax-Types: Number Literals

## Summary
This RFC proposes adding support for number literal types to Luau, allowing developers to define types that represent specific numeric values rather than the entire number domain.
This would enable more precise type checking for functions and variables that should only work with specific numeric constants.

## Motivation
In many programming scenarios, we need to work with specific numeric values that have special meaning in our code. For example, HTTP status codes, animation states, or configuration constants often have specific numeric values that carry semantic meaning. Currently, Luau only allows us to type these as `number`, which doesn't capture the intent that only specific values are valid.

Some use cases include:

1. Functions that only accept specific numeric arguments (see below)
2. Union types of allowed numeric constants
3. Improved type safety for numeric enums
4. More precise return type definitions for functions that return specific numeric values

By adding number literal types, we would be able to express these constraints at the type level, catching more errors at compile-time rather than runtime.

## Design

### Syntax
Number literal types will use the same syntax as number literals in value expressions. Any numeric literal can be used as a type:

```lua
local statusOK: 200 = 200
local pi: 3.14159 = 3.14159
local negativeOne: -1 = -1
```

These can be combined with union types to represent a set of allowed values:

```lua
type HttpSuccessCode = 200 | 201 | 202 | 204
type Direction = 0 | 90 | 180 | 270
```

And used in function signatures:

```lua
local function setVolume(level: 0 | 1 | 2 | 3 | 4 | 5): boolean
    -- Only accepts volume levels 0-5
end

local function isEven(n: number): 0 | 1
    return n % 2
end
```

### Type Compatibility

A number literal type is compatible with the `number` type, but not vice versa. This means:

```lua
local x: number = 5    -- OK
local y: 5 = x         -- Type error: 'number' is not compatible with '5'
local z: 5 = 5         -- OK
local w: 5 = 6         -- Type error: '6' is not compatible with '5'
```

When used with union types, the compatibility follows expected rules:

```lua
type Dice = 1 | 2 | 3 | 4 | 5 | 6

local roll: Dice = 3           -- OK
local badRoll: Dice = 7        -- Type error: '7' is not compatible with 'Dice'
local anyNumber: number = roll -- OK
```

Right now, you would have to `tostring` the status code, and that would not be possible for union types without a massive huge wrapper. This helps with the complexity of code, and allows for cleaning it up.

### Type Inference

The type checker will infer the most specific type when dealing with number literals:

```lua
local x = 42       -- Inferred as '42', not 'number'

local function double(n)
    return n * 2
end

local y = double(5) -- Inferred as 'number', not '10'
```

However, to avoid making the type system too rigid, arithmetic operations on number literal types will generally result in the `number` type:

```lua
local x: 5 = 5
local y: 3 = 3
local z = x + y    -- Inferred as 'number', not '8'
```

### Generic Constraints

Number literal types can be used as generic constraints:

```lua
local function createArray<T, N: number>(value: T, length: N): {T}
    local arr = {}
    for i = 1, length do
        arr[i] = value
    end
    return arr
end

local arr1 = createArray("hello", 3)  -- OK: Creates array of length 3
local arr2 = createArray("world", -1) -- Type error: Negative length
```

## Drawbacks

1. ***Increased Complexity***: Adding number literal types increases the complexity of the type system, which might make it harder for new users to understand.
2. ***Performance***: The type checker will need to handle more specific types, which could potentially impact performance for large codebases.
3. ***Studio Integration***: The autocomplete might need updates to properly handle the new type information.
4. ***Type Explosion!*** Developers might create excessive unions of number literal types, leading to verbose type annotations.
5. ***Limited Usefulness***: In many cases, using enums or string literal types might be more appropriate and readable than number literal types.

## Alternatives

### Just don't add number literal types
We could continue using the current approach where specific numbers are just typed as `number`. Developers would need to add runtime checks for specific values.

### Potentially, custom nominal types for numeric constants
Instead of allowing any number as a type, we could introduce a special syntax for defining named numeric constants:

```lua
type HttpStatus = @enum {
    OK = 200,
    CREATED = 201,
    -- etc.
}
```

This would be a lot more explicit but a little less flexible than number literal types.

### Add support for numeric ranges
Instead of just supporting specific numeric values, we could support ranges:

```lua
type Percentage = 0..100
```

This would be more powerful but (assumingly) significantly more complex to implement.
