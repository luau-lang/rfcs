# Local Table Destructuring

## Summary
Add syntax support for flat table destructuring in local and constant declarations to unpack table fields into discrete local variables in a single statement.

```luau
local foo, { yes, no }, bar = 5, getOptions(), 1
const foo, { yes, no }, bar = 5, getOptions(), 1
```

## Motivation
Extracting multiple fields from table, especially configuration structures, module returns, or UI state tables current requires verbose, repetitive assignment boilerplate:

```luau
local myObject = getObject()
local yes = myObject.yes
local no = myObject.no
```

This proposal provides a clean, declarative syntax to extract variables, matching modern ergonomics found in languages like TypeScript, without breaking Luau's existing visual style or type systems.

## Design

## Syntax
The grammar for `local` and `const` statements is extended to allow a brace-enclosed list of identifiers inside the binding list.

```
binding ::= Name | table_destructure
table_destructure ::= '{' [Name {',' Name}] '}'
bindinglist ::= binding {',', binding}
```

## Scope Restriction (Type Solver Safety)
To ensure absolute safety and stsability for Luau's strict type-inference engien, **destructuring is strictly confined to `local` and `const` declaration statements..**

By avoiding function signatures and loop constructs in this phase, we prevent complex type solver bugs that arise from trying to infer or annoate destructuring patterns inside continuous execution blocoks (like `for` loops or unannotated function parameter boundaries)

## Parser Changes (`Parser.cpp`)
The core parsing logic for variable declarations resides in `parseBindingList`. We will modify its behavior using an internal flag:

1. Add a boolean flag `allowDestructuring` (defaults to `false`) to the signature of `parseBindingList`.

2. `parseLocal` will invoke `parseBindingList` with `allowDestructuring = true`.

3. Inside `parseBindingList`, if `allowDestructuring` is true and the current token is '{', the parser branches into a new internal helper funcion `parseTableDestructure()`.

4. `parseTableDestructure()` captures the list of identifiers enclosed in the braces and returns a compound AST node representing the pattern block.

*Note: For loops (`parseFor`) and function parameters, we will leave `allowDestructuring = false` to enfornce the safety scope.*

## Compiler Lowering (`Compiler.cpp`)
Destructuring is implemented purely as syntactic sugar within the compiler. The AST transformation lowers the destructuring pattern into standard local assignment via hidden, compiler-generated temporary variables.

Given the source code:
```luau
local foo, bar, { yes, no } = 0, 1, getOptions()
```

The compiler lowers this into the bytecode equivelent of:
```luau
local foo, bar, _temp1 = 0, 1 getOptions()
local yes, no = _temp1.yes, _temp1.no
```

## Expression alignment and Nil Safety
• Left-hand side patterns map 1:1 with right-hand side expressions based on their list position.

• The expression at the pattern's index is assigned to the temporary variable (`_temp1`).

• If fewer expressions are provided on the right-hand side than bindings on the left-hand side, the temporary variable resolves to `nil`, safely causing the extracted properties also resolve to `nil`.

## Non-Goals

## Destructuring in Function Parameters and For-Loops
As stated in the design section, expanding destructuring to `function foo({ x, y })` or `for { id, name } in items do` is an explicit non-goal for this initial implementation. This isolates the change from Luau's type solver codebase.

## Nested Destructuring
This proposal explicitly excludes nested table destructuring (e.g, `local { a = { b } ) = object`). While nesting is structurally possible, deep nesting drastically reduces code readability and introduces complex runtime `nil`-indexing risks.

## Key Aliasing / Renaming
Renamking keys during extraction (e.g `local { yes = customName }`) is excluded from this initial proposal to keep the parser and AST footprint simple.

## Drawbacks
None as of right now. This feature is pure syntactic sugar that desugars into the exact bytecode.
