# Key destructuring

## Summary

Introduce a new syntax for unpacking key values into their own variables, such that:

```lua
local { .a, .b } = t
-- a == t.a
-- b == t.b
```

## Motivation

Simple indexes on tables are very common both in and outside of Luau. A common use case is large libraries. It is common in the web world to see something like:

```js
const { useState, useEffect } = require("react");
```

...which allows you to quickly use `useState` and `useEffect` without fully qualifying it in the form of `React.useState` and `React.useEffect`. In Luau, if you do not want to fully qualify common React functions, the top of your file will often look like:

```lua
local useEffect = React.useEffect
local useMemo = React.useMemo
local useState = React.useState
-- etc
```

...which creates a lot of redundant cruft.

It is also common to want to have short identifiers to React properties, which basically always map onto a variable of the same name. As an anecdote, a regex search of `^\s+local (\w+) = \w+\.\1$` comes up 103 times in the My Movie codebase, many in the form of indexing React properties:

```lua
local position = props.position
local style = props.style
-- etc...
```

...whereas in JavaScript this would look like:
```js
const { position, style } = props

// Or even...
function MyComponent({
	position,
	style,
})
```

React properties are themselves an example of a common idiom of passing around large tables as function arguments, such as with HTTP requests:

```js
// JavaScript
get("/users", ({
	users,
	nextPageCursor,
}) => { /* code */ })
```

## Design

This RFC proposes expanding the grammar of `binding`:
```patch
binding = NAME [':' Type]
+         | `{` keydestructor [`,` keydestructor] [`,`] `}` [':' Type]

+keydestructor = `.` NAME [':' Type]
```

This would allow for the following:

```lua
local { .a, .b }, c = t

for _, { .a, .b } in ts do
end

local function f({ .a, .b }, c)
end
```

In all of these cases, `.x` is an index of the table it corresponds to, assigned to a local variable of that name. For example, `local { .a, .b } = t` is syntax sugar for:

```lua
local a = t.a
local b = t.b
```

This will have the same semantics with regards to `__index`, in the order of the variables being assigned. Furthermore, if `t` for whatever reason cannot be indexed (not a table, nil), you will get the same errors as you would if you were writing out `t.a` by hand.

Trying to use object destructuring in a local assignment without a corresponding assignment, such as...
```lua
local { .x, .y }
```

...is valid to parse and will execute, but can be linted against. This might even be emergent, as this will be statically equivalent to `nil.x`, which Luau's type system can catch.

#### Types
An optional type can be supplied, such as:
```lua
local { .a: string, .b: number } = t

-- Equivalent to...
local a: string = t.a
local b: number = t.b
```

Without explicit types, local assignments and for loops will assume the type of `<rhs>.<field>`. For example...
```lua
-- x and y will both be typed `number` here
local { .x, .y } = position :: { number }
```

As a function argument, it will act the same as if writing it out by hand, where anonymous types are created:
```lua
-- Will be f({+ x: a, y: b +})
local function f({ .x, .y })
```

Additionally, you can specify a type on the "table" as a whole.

```lua
local { .x, .y }: Position = p
```

Combining both is acceptable, in which case the type on the variable takes priority:

```lua
-- `x` will be `number`, `y` will be the type of `T.y`
local { .x: number, .y }: T = p
```

### Non-designs
#### Renaming variables
This proposal does not allow something like JavaScript's:
```js
const { a: b } = t
```

...where `t.a` is bound to `b`. This is not undesired, but this RFC is not concerned with it for now. This also means that any field you want to destruct must be a valid Luau identifier.

#### Arrays
This RFC does not concern itself with array destructuring. This is in part because it is not obvious what a reasonable syntax for it would be, and because its purposes might be better suited by tuples.

#### Reassignments
We do not support key destructuring in reassignments, for example...

```lua
{ .x, .y } = position
```

This is to avoid ambiguity with potential table calls:

```lua
local a = b
{ .x, .y } = c
```

While with only this RFC, this is unambiguous with 2 lookahead (`{` + `.`), it could potentially complicate our ability to expand the key destructuring syntax, such as to add renaming variables.

## Drawbacks

### Roblox - Property casing
Today in Roblox, every index doubly works with camel case, such as `part.position` being equivalent to `part.Position`. This use is considered deprecated and frowned upon. However, without (or even with) variable renaming, this becomes significantly more appealing. For example, it is common you will only want a few pieces of informaiton from a `RaycastResult`, so you might be tempted to write:

```lua
local { position } = Workspace:Raycast(etc)
```

...which would work as you expect, but rely on this deprecated style.

## Alternatives

### Syntax

Many syntaxes have been proposed for destructuring. The most significant problem with any proposal is that is must be unambiguous to the reader whether or not the destructor is for **dictionaries** or for **arrays**.

An intuitive suggestion is `local { a, b } = t`, but this syntax fails this test--it is not obvious if this is `local a = t.a` or `local a = t[1]`, regardless of whatever syntax is chosen for array destructuring (should it exist).

### Focus more precisely, rather than using binding
We could only allow this more precisely, such as just on local assignments. This would satisfy the use case of library importing and React properties (partially, since it would be its own line), but this would limit the use case of tables as arguments to functions, such as in the motivating HTTP request example.
