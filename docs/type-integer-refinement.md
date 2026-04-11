# Signed and Unsigned Integer Subtype Refinements

## Summary
This proposal introduces `signed` and `unsigned` as subtype refinements of the existing `integer` primitive type. By establishing a type hierarchy where `signed <: integer` and `unsigned <: integer`, the Luau type checker can distinguish between integers intended for signed operations (e.g., `integer.lt`) and those intended for unsigned operations (e.g., `integer.ult`). This allows the `integer` library to enforce type safety at the type-checking level, preventing the accidental use of signed integers in unsigned contexts and vice versa.

## Motivation
With the introduction of the `integer` primitive and the `integer` library, Luau provides powerful tools for low-level integer manipulation. However, a critical gap exists: there is currently no way to communicate or validate whether an integer is signed or unsigned within the type system.

Currently, a user could pass a negative integer to `integer.ult` (Unsigned Less Than), and the type checker would not flag this as an error because both arguments are simply typed as `integer`. This shifts the burden of validation entirely to the developer or results in unexpected runtime behavior.

By introducing subtype refinements, we can:
1. **Prevent Logic Errors:** Catch cases where a signed value is passed to an unsigned function before the code ever runs.
2. **Improve Documentation:** Type signatures for library functions become self-documenting (e.g., seeing `unsigned` immediately tells the developer the expected range).
3. **Reduce Runtime Overhead:** By validating signedness at the type-checking stage, we reduce the need for expensive runtime assertions within the `integer` library.

## Design
We propose the introduction of two new type refinements: `signed` and `unsigned`. These are not separate primitive types, but rather specialized versions of the `integer` type.

### Type Hierarchy
The relationship is defined as:
- `signed` as subtype of `integer`
- `unsigned` as subtype of `integer`

### Integration with the Integer Library
The `integer` library functions will be updated to accept these refined types. To ensure a smooth transition and avoid unnecessary casting for general-purpose integers, functions will accept either the refined subtype OR the base `integer` type. When the base `integer` type is provided, the type checker will implicitly refine it to the required subtype for that operation.

**Example Signatures:**
- `integer.lt(a: signed, b: signed): boolean`
- `integer.ult(a: unsigned, b: unsigned): boolean`

### Behavior and Examples

#### 1. Implicit Refinement
If a variable is typed as the general `integer` type, it can be used in either signed or unsigned functions. The type checker treats this as a valid "promotion" to the required refinement.

```luau
local x: integer = 10i
local y: integer = 20i

integer.lt(x, y)  -- Valid: integer refined to signed
integer.ult(x, y) -- Valid: integer refined to unsigned
```

#### 2. Type Enforcement
If a variable is explicitly typed as `signed` or `unsigned`, the type checker will prevent it from being used in a function requiring the opposite refinement.

```luau
local s: signed = -5i
local u: unsigned = 5i

integer.lt(s, s)  -- Valid
integer.ult(s, s) -- Type Error: Expected unsigned, got signed

integer.ult(u, u) -- Valid
integer.lt(u, u)  -- Type Error: Expected signed, got unsigned
```

#### 3. Function Overloads/Refinement
Functions can be written to accept the base `integer` type and refine it internally, or specify the refinement to constrain the input.

```luau
-- Expects only the 'unsigned' integer subtype
local function processUnsigned(val: unsigned)
    print(integer.ult(val, 100i))
end

-- Accepts 'integer' and its subtypes
local function processInteger(val: integer)
    -- Type errors if val was passed as 'signed' integer subtype 
    print(integer.ult(val, 100i)) -- 'integer' refines to 'unsigned'
end

local a: integer = 10i
local b: integer = -10i

processUnsigned(a)    -- Type Error: Expected unsigned, got integer
processInteger(b)     -- Valid ('integer' refines to 'unsigned')
```

## Drawbacks
- **Type System Complexity:** Introducing refinements adds another layer of complexity to the Luau type checker and may increase the learning curve for new users.
- **Casting Requirements:** Users who intentionally want to treat a `signed` integer as `unsigned` (e.g., for bit-casting purposes) will now be required to use a type cast (e.g., `x :: unsigned`), which may be seen as verbose.
- **Incremental Adoption:** Existing codebases using the `integer` library may see a surge in type errors if the library signatures are updated, though the "implicit refinement" of the base `integer` type is intended to mitigate this.

## Alternatives
- **Separate Primitive Types:** We could introduce `uint` and `int` as entirely separate types. However, this would be too rigid, as it would break the existing `integer` primitive and force users to constantly cast between the two for basic arithmetic.
- **More Verbose Subtype Naming:** `signedinteger`/`unsignedinteger` avoids ambiguity, but would result in the longest type names in Luau. Casting would look like `(value :: signedinteger)`, and it should be considered that `signed`/`unsigned` would co-exist in proximity of `integer` library usage.
- **Runtime Validation:** We could add runtime checks to every `integer` library function. This would catch errors but would introduce a performance penalty to the very library designed for high-performance integer operations.
- **Do Nothing:** Maintain the current state. The impact would be a continued risk of signed/unsigned mismatch bugs and a lack of type-level clarity in the Luau ecosystem.
