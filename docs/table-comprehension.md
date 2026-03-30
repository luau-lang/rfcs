
# Table Comprehension

## Summary

Add table comprehension syntax for constructing arrays and maps using inline iteration and optional filtering expressions.

---

## Motivation

Constructing tables using loops is a very common pattern in Luau. However, this often results in verbose and mutation-heavy code.

For example:

```lua
local result = {}
for _, v in ipairs(items) do
    if v > 1 then
        table.insert(result, v * 2)
    end
end
````

This approach has several drawbacks:

* Requires explicit mutation (`table.insert`)
* Introduces boilerplate for simple transformations
* Obscures intent for simple mapping/filtering operations
* May be harder for the compiler to optimize

A table comprehension syntax would allow this to be written more concisely:

```lua
local result = {v * 2 for _, v in ipairs(items) if v > 1}
```

This pattern is widely used in other languages (e.g. Python) and Lua-adjacent languages (e.g. MoonScript), demonstrating its usefulness for data transformation.

---

## Design

### Syntax

Two forms of table comprehension are proposed:

#### Array-style comprehension

```lua
{expression for variables in iterator [if condition]}
```

Examples:

```lua
local doubled = {v * 2 for _, v in ipairs(items)}
local filtered = {v for _, v in ipairs(items) if v > 1}
```

---

#### Map-style comprehension

```lua
{[key] = value for variables in iterator [if condition]}
```

Examples:

```lua
local map = {[k] = v for k, v in pairs(t)}
```

---

### Grammar

The following modification to the Luau grammar is proposed:

```diff
tableconstructor ::= '{' [fieldlist | comprehension] '}'

+ comprehension ::= exp 'for' namelist 'in' explist ['if' exp]
```

---

### Semantics

#### Evaluation order

* The iterator expression is evaluated first
* The comprehension executes similarly to a `for` loop
* The optional `if` condition filters elements

---

#### Array behavior

```lua
{expr for _, v in ipairs(t)}
```

is equivalent to:

```lua
local result = {}
for _, v in ipairs(t) do
    result[#result + 1] = expr
end
```

---

#### Map behavior

```lua
{[k] = v for k, v in pairs(t)}
```

is equivalent to:

```lua
local result = {}
for k, v in pairs(t) do
    result[k] = v
end
```

If duplicate keys are produced, the last assignment takes precedence.

---

#### Scope

* Variables introduced in the comprehension are scoped to the comprehension
* They are not visible outside the expression

---

#### Nil handling

If the comprehension expression evaluates to `nil`, the value is still inserted into the table (consistent with standard table behavior).

---

### Type Semantics

* Array comprehensions infer type `{T}` where `T` is the type of the expression
* Map comprehensions infer type `{[K]: V}` based on key and value expressions
* The type checker should treat comprehensions similarly to their desugared loop equivalents

---

### Language implementation

This feature can be implemented entirely in the compiler by lowering comprehensions into equivalent `for` loops.

For example:

```lua
{expr for _, v in ipairs(t)}
```

can be compiled into:

```lua
local result = {}
for _, v in ipairs(t) do
    result[#result + 1] = expr
end
```

This approach:

* Requires no changes to the Luau VM
* Reuses existing loop semantics
* Allows for future optimizations (e.g. preallocation)

---

## Drawbacks

* Adds new syntax to the language
* May be misused in complex or nested forms, reducing readability
* Not strictly necessary, as equivalent functionality already exists using loops

---

## Alternatives

### Existing loop-based approach

```lua
local result = {}
for _, v in ipairs(items) do
    table.insert(result, v * 2)
end
```

This is more verbose and mutation-based.

---

### Helper functions

Users may define utility functions for mapping/filtering, but these introduce additional abstraction and may reduce performance or clarity.

---

## Prior Art

* Python list/dict comprehensions
* MoonScript table comprehensions

````
