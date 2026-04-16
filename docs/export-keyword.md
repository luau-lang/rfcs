# Export by Value

## Summary
Extend the `export` keyword to support variables and functions as syntax sugar for constructing a table and returning it from a module.

## Motivation
Today, type aliases are able to be exported from a module using the `export` keyword.

```luau
export type Point = {x: number, y: number}
```

However, this mechanism is currently only supported by types.

Extending  `export`  to also support variable and function declarations would provide a consistent way for users to expose a stable API from modules.

Furthermore, introducing a language-level concept of exporting opens the door to many optimizations not currently possible with dynamic module returns, such as cross-module inlining and constant-folding.

## Design
The `export` contextual keyword will now be allowed anywhere before variable and function declarations at the top level of a module, including `local`, `const` and `function` declarations.

Exporting declarations that are in nested `do end` blocks under the top scope are also permitted. Attempting to export a declaration outside the module scope will result in a parse error.

Exported variables will count towards the local variable limit, as they can be optimized into locals by the compiler.

```luau
export local version = "5.1"
export const TAU = math.pi * 2

export function init() -- exported functions are always const
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
	export local bar = 1 -- not allowed, syntax error
end
```

Just like normal variable declarations, it is possible to export multiple variables, variables with type annotations, uninitialized variables, and const variables.

```luau
export local settings: Settings = getSettings()
export local a, b, c = 1, 2, 3
export local d -- same as export d = nil
export const MAX_ITEMS = 10 -- cannot be reassigned
```

Exporting a variable with the same identifier twice is a parse error, regardless of the scope it was defined in.

```luau
export local foo = 1
do
	export local foo = 2 -- syntax error
end
```

However, exporting a variable with the same identifier as another non-exported variable is allowed, following conventional lexical scoping and shadowing rules.

```luau
local function foo() return 1 end
export function foo() return 2 end -- exported, shadows foo

print(foo()) -- 2

local fruit = "apple"
export local fruit -- exports fruit = nil, not "apple"
print(fruit) -- nil

export local animal = "dog"
local animal = "cat" -- shadows animal, doesn't change export
animal = "bird"

print(animal) -- bird
```

Exported function declarations are also always treated as const, as reassigning function declarations is usually a mistake.

```luau
export function f() end
f = 1 -- illegal
```

### Desugared Form
Exporting variables desugars into assigning keys to a table that is then frozen and returned once the module scope ends.

```luau
export local a = 1

-- desugars into

local _EXP = {}
_EXP.a = 1
return table.freeze(_EXP)
```

Exported local variables can therefore be reassigned within the module after being declared. This allows for conditional exports and forward declarations.

```luau
export local side = "heads"
if math.random(0, 1) == 1 then
	side = "tails"
end

export local f, g
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

Once the module ends and the export table is frozen, subsequent reassignments will throw a runtime error, analogous to reassigning keys to a frozen table.

```luau
export local counter = 0

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

### Typechecking
Type inference of exported variables will behave exactly the same as local variable inference.

```luau
-- example behavior subject to change with local inference changes

export local foo = 15
foo = "hello"
foo = true
-- foo: number | string | boolean

export local bar = nil
if math.random(0, 1) == 1 then
	bar = 123
end
-- bar: number?
```

This is in contrast to how typechecking would behave in the desugared form with key assignments.

```luau
local _EXP = {}
_EXP.foo = 15
_EXP.foo = "hello" -- type error
_EXP.foo = true -- type error

_EXP.bar = nil
if math.random(0, 1) == 1 then
	_EXP.bar = 123 -- type error
end

return table.freeze(_EXP)
```

This is done to ensure exported variables act exactly the same as normal variables and preserve the same ergonomics.

### Interaction with `return`

A module that contains an export statement is not permitted to also contain a return statement at the module scope, as the two mechanisms are mutually exclusive. If there is both an export statement and return statement, it is a parse error.

```luau
export local a = 1
return {b = 2} -- syntax error
```
```luau
if skip then return end
export local a = 1 -- syntax error
```

This restriction does not apply to modules that only contain type exports for backwards compatibility.

### Future Optimizations

While the primary purpose for extending exports is user ergonomics, first-class exports also open the door to many optimizations that aren't currently possible with dynamic module returns.

For example, exported variables and functions can be transformed into local variables that are stored in VM registers for faster lookup:

```luau
export const TAU = math.pi * 2
print(TAU)

-- becomes
const TAU = math.pi * 2
print(TAU)
return table.freeze({TAU = TAU})
```

Furthermore, first-class exports allow for the compiler to assume exported variables are never reassigned after module return, unlocking the ability to bring constant-folding and inlining across module boundaries.

## Drawbacks

This increases compiler complexity in terms of tracking exported declarations and converting them into table assignments.

Allowing reassignment of exported keys after module return isn't possible. This is a deliberate design choice intended to make exports easier to optimize.

The shadowing rules for exported variables with relation to non-exported variables may lead to confusion. Similarly, reassigning exported variables in a large file may be as equally confusing. Users are encouraged to declare their exported variables with const if they do not intend to reassign them in the module.

Uninitialized exported variables pose a footgun where one can declare an uninitialized variable that is never reassigned to. This can be solved with linting.

## Alternatives

As always, do nothing and leave users to construct their own module return tables. This would forfeit providing a consistent way for users to expose public APIs and would not allow for future cross-module optimizations.

Permitting exports in `do end` blocks can be scrapped, but this would prevent users from scoping exported declarations and doesn't make much sense not to support. It is also possible to initially not support this and then add it back in later.

An earlier version of the exported variable syntax omitted declaration keywords, such as with `export x = 1`. This was changed in lieu of the addition of const.

Exported functions may instead not be automatically const, or declaration keywords may be allowed before exported functions such as with `export const function f() end`. Neither of these options are done since reassigning function declarations is already discouraged practice, and adding more keywords between function exports makes it unnecessarily verbose.

All exported variables could be treated as const with applicable syntax changes. However, this makes many useful patterns such as exporting mutually dependent functions hard to pull off. Furthermore, introducing categorical const-ness would not allow for any more optimization opportunities.
