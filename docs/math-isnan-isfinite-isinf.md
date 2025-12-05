# math.isnan, math.isinf and math.isfinite for Math Library

**Status**: Implemented

## Summary

This RFC proposes a `math.isnan`, a `math.isinf`, and a `math.isfinite` function to be added to the math library.
These functions will validate that numbers are not NaN and are "finite" (non-NaN non-inf valid numbers) 
respectively.

## Motivation

Currently, determinating if a value is `NaN` requires non-obvious and non-semantic check `x ~= x`. Determining if
a number is finite requires a verbose check that handles both `NaN` and `Inf` cases, typically like `x == x` 
& `math.abs(x) ~= math.huge`. Checking specifically for Infinity also requires a manual check like 
`math.abs(x) == math.huge`.

Adding explicit `math.isnan(x)`, `math.isinf(x)` and `math.isfinite(x)` functions brings Luau's math library 
in line with modern languages (such as C++, JavaScript, Python, etc), improving code clarity, robustness and 
safety for mathemetical applications. In networked environments, validating numbers received from the client is a 
must and applicable across most games.

## Design

We propose adding three new functions to the global `math` table:


`math.isnan(x: number): boolean`

Returns `true` if `x` is Not-a-Number (`NaN`), and false otherwise.

**Example:**
```luau
local x = 0 / 0 -- produces NaN
local y = 10

print(math.isnan(x)) -- true
print(math.isnan(y)) -- false
```

`math.isinf(x: number): boolean`


Returns `true` if `x` is positive or negative infinity (`Inf`), and `false` otherwise.

**Example:**
```luau
local x = math.huge
local y = 10

print(math.isinf(x)) -- true
print(math.isinf(y)) -- false
print(math.isinf(0 / 0)) -- false (NaN)

```


`math.isfinite(x: number): boolean`

Returns `true` if `x` is a finite number, meaning it is neither `NaN` nor positive or negative infinity 
(`math.huge`).

**Example:**
```luau
local x = math.huge
local y = 0 / 0
local z = 123.45

print(math.isfinite(x)) -- false
print(math.isfinite(y)) -- false
print(math.isfinite(z)) -- true
```


## Drawbacks

1. Increased API: This proposal adds three new functions to the global `math` library, increasing the overall
   size of the standard library.
2. Redundancy for experienced developers: Experienced developers familiary with these checks, may view these
   functions as unnecessary syntactic sugar for the already fast and idiomatic check `x ~= x` and
   `math.abs(x) ~= math.huge`.

## Alternatives

### **Alternative 1: Status Quo**

**Proposal:** Do not add new functions. Continue to rely on checks for both.

**Rejection:** This sacrifices clarity and developer experience for the sake of library minimalism. The lack of 
standard functions makes code more brittle and harder to read, especially for newcomes to Luau.

### **Alternative 2: Only add `math.isfinite`**

**Proposal:** Only add the primary utility check, `math.isfinite(x)`, as it covers the most common use case 
(checking if a number is safe). Users could rely on idioms (`x ~= x`) for `NaN` and (`math.abs(x) == math.huge`) 
for `Inf`.
