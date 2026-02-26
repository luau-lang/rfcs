# Export-by-Value

## Summary
Extend the `export` keyword to support values and functions as syntax sugar for constructing a table and returning it from a module.

## Motivation
Today, type aliases are able to be exported from a module using the `export` keyword.

```luau
export type Point = {x: number, y: number}
```

However, this mechanism is currently limited to types.

Extending  `export`  to support values and functions would provide a consistent way for users to expose a stable API from a module. Furthermore, first-class module exports can provide future static optimization opportunities, such as cross-module inlining and constant folding.

## Design
Allow the `export` contextual keyword anywhere the `local` keyword may be used at the top level of a module, as well as within nested `do end` blocks.

Exported variables will count towards the local variable limit, as they can be optimized into locals by the compiler.

```luau
export version = "5.1"

export function init()
	-- TODO
end

do
	local counter = 0
	
	export function increment(): number
		counter += 1
		return counter
	end
end
-- counter and increment not visible here

if foo then
	export bar = 1 -- not allowed, syntax error
end
```

Just like local variable declarations, it is possible to export multiple variables, variables with type annotations, and uninitialized variables.

```luau
export settings: Settings = getSettings()
export a, b, c = 1, 2, 3
export d -- same as export d = nil
```

Exporting a variable with the same identifier twice is a syntax error, regardless of the scope it was defined in.

```luau
export foo = 1
do
	export foo = 2 -- syntax error
end
```

However, exporting a variable with the same identifier as another non-exported variable is allowed, following conventional lexical scoping and shadowing rules.

```luau
local function foo() return 1 end
export function foo() return 2 end -- exported, shadows foo

print(foo()) -- 2

local fruit = "apple"
export fruit -- exports fruit = nil, not "apple"
print(fruit) -- nil

export animal = "dog"
local animal = "cat" -- shadows animal, doesn't change export
animal = "bird"

print(animal) -- bird
```

### Desugared Form
Exporting variables desugars into assigning keys to a table that is then frozen and returned once the module scope ends.

```luau
export a = 1

-- desugars into

local _EXP = {}
_EXP.a = 1
return table.freeze(_EXP)
```

Exported values can therefore be reassigned within the module after being declared. This allows for conditional exported values and forward declarations.

```luau
export side = "heads"
if math.random(0, 1) == 1 then
	side = "tails"
end

export f, g
function f()
	g()
end
function g()
	f()
end

-- desugars into

local _EXP = {}
_EXP.side = "heads"
if math.random(0, 1) == 1 then
	_EXP.side = "tails"
end

_EXP.f, _EXP.g = nil, nil
function _EXP.f()
	_EXP.g()
end
function _EXP.g()
	_EXP.f()
end

return table.freeze(_EXP)
```

Once the module ends and the export table is frozen, subsequent reassignments will throw a runtime error, analogous to attempting to assign a key in a frozen table.

```luau
export counter = 0

export function increment()
	-- once the module returns
	-- this raises an "attempt to modify a readonly table" error
	counter += 1
end

-- desugars into

local _EXP = {}
_EXP.counter = 0

function _EXP.increment()
	_EXP.counter += 1
end

return table.freeze(_EXP)
```

### Interaction with `return`

A module that contains an export statement is not permitted to also contain a return statement at the module scope, as the two mechanisms are mutually exclusive. If there is both an export statement and return statement, it is a syntax error.

```luau
export a = 1
return 2 -- syntax error
```
```luau
if skip then return end
export a = 1 -- syntax error
```

This restriction does not apply to modules that only contain type exports for backwards compatibility.

### Future Optimizations

While static first-class exports primarily provide improvements to user ergonomics, they also open the door for many optimizations that aren't currently possible with dynamic module returns.

For example, exported variables and functions could be transformed into local variables that are stored in VM registers for fast lookup:

```luau
export tau = math.pi * 2
print(tau)

-- becomes
local tau = math.pi * 2
print(tau)
return table.freeze({tau = tau})
```

Furthermore, static exports that are never reassigned are capable of being inlined and constant folded across modules, given future static imports.

## Drawbacks

This increases compiler complexity in terms of tracking exported tokens and converting them into table assignments.

Since `export` is a contextual keyword, the following statements are valid and might cause confusion. These are solved by current linting and typechecking.

```luau
export = 1
export(1)
export "a"
export {x = 1}
```

Similarly, future RFCs such as a theoretical table destructing RFC may run into syntax ambiguities with export declarations, such as below. The first option could still be parsed with lookahead, but the second option would not be possible.

```luau
export {a, b} = t -- looks like export({a, b}) = t
export [a] = t -- looks like export[a] = t
```

## Alternatives

As always, do nothing and leave users to construct their own module return tables. This would forfeit providing a consistent way for users to expose a public API and would not allow for future cross-module optimizations.

Exporting in `do end` blocks under the module scope can be scrapped, but this would prevent users from lexically scoping exported variables and doesn't make much sense not to support. It is also possible to initially not support this and then add it back in later.

Exported variables could be treated as unassignable after initialization (like `const` variables in JavaScript), but this would make useful patterns such as exporting mutually dependent functions hard to pull off. Furthermore, introducing const-ness to the language would not provide any further optimization opportunities, as the compiler can already determine if a variable is constant if it is never reassigned to.

So-called "shorthand exports" could also be introduced to allow exporting local variables after they've been declared. This only makes sense under the model that exports are unassignable and introduces confusing semantics regarding variable aliasing.
