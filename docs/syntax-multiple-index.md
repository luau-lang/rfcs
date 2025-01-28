# Multiple indexing

## Summary

Introduce multiple indexing, an extension of table indexing that allows for multiple values to be read at the same time, with dedicated array and map shorthands for ergonomics.

This allows for destructuring that:
- works with array & map style tables
- shortens both declaration and assignment
- doesn't require parser backtrack
- is consistent with Luau style

## Motivation

This is intended as a spiritual successor to the older ["Key destructuring" RFC by Kampfkarren](https://github.com/luau-lang/rfcs/pull/24), which was very popular but was unfortunately not able to survive implementation concerns.

----

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

// Supported in JavaScript, but not this proposal
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

## Structure matcher

This proposal will use the term *structure matcher* to refer to syntax for retrieving values from table structures. 

Structure matchers can appear:
- In place of the identifiers in a `local ... = ...` declaration statement
- In place of the identifiers in a `... = ...` assignment statement

A structure matcher starts with the `in` keyword, followed by braces. The keyword is necessary to avoid ambiguity on the LHS of assignments.

```Lua
local in { } = data
in { } = data
```

### Matching using dot indexing

Luau inherits the "dot indexing" shorthand, allowing string keys to be easily indexed:

```Lua
local foo, bar = data.foo, data.bar
```

In structure matchers, identifiers can be specified with a dot prefix in a similar fashion.

The identifier acts both as the bound variable name, and as the index to use.

```Lua
local in { .foo, .bar } = data
in { .foo, .bar } = data
```

This desugars to:

```Lua
local foo, bar = data.foo, data.bar
foo, bar = data.foo, data.bar
```


## Alternatives

### Braces around identifier list without prefix

The previously popular RFC used braces around the list of identifiers to signal destructuring, and dot prefixes to disambiguate array and dictionary destructuring:

```Lua
local rootUtils = require("../rootUtils")
local { .homeDir, .workingDir } = rootUtils.rootFolders
```

One reservation cited would be that this is difficult to implement for assignments without significant backtracking:

```Lua
local rootUtils = require("../rootUtils")
{ .homeDir, .workingDir } = rootUtils.rootFolders
```

Removing the braces and relying on dot prefixes is not a solution, as this still requires significant backtracking to resolve:

```Lua
local rootUtils = require("../rootUtils")
.homeDir, .workingDir = rootUtils.rootFolders
```

It also does not provision for destructuring in the middle of an expression, which would be required for fully superseding library functions such as `table.unpack`. This would leave Luau in limbo with two ways of performing an unpack operation, where only one is valid most of the time.

As such, this proposal does not pursue these design directions further, as the patterns it proposes struggle to be extrapolated and repeated elsewhere in Luau.

### Indexing assignment

To address the problems around assignment support, a large amount of effort was poured into finding a way of moving the destructuring syntax into the middle of the assignment.

A `.=` and/or `[]=` assignment was considered for this, for maps and arrays respectively:

```Lua
local amelia, bethany, caroline .= nicknames
local three, five, eleven []= numbers
```

However, this was discarded as it does not align with the design of other compound assignment operations, which mutate the left-hand-side and take the right-hand-side of the assignment as the right-hand-side of the operation itself.

```Lua
local foo = {bar = "baz"}
foo .= "bar"
print(foo) --> baz
```

Many alternate syntaxes were considered, but discarded because it was unclear how to introduce a dinstinction between maps and arrays. They also didn't feel like they conformed to the "shape of Luau".

```Lua
local amelia, bethany, caroline [=] nicknames
local amelia, bethany, caroline ...= nicknames
local ...amelia, bethany, caroline = nicknames
```

### Type-aware destructuring

Another exploration revolved around deciding between array/map destructuring based on the type inferred for the right-hand-side.

However, this was discarded because it made the behaviour of the assignment behave on non-local information, and was not clearly telegraphed by the syntax. It would also not work without a properly inferred type, making it unusable in the absence of type checking.

### Multiple indexing

A syntax for indexing multiple locations in a table was considered, but rejected by the Luau team over concerns it could be confused for multi-dimensional array syntax.

```Lua
local numbers = {3, 5, 11}

local three, five, eleven = numbers[1, 2, 3]
```

### Don't do anything

This is always an option, given how much faff there has been trying to get a feature like this into Luau!

However, it's clear there is widespread and loud demand for something like this, given the response to the previous RFC, and the disappointment after it was discarded at the last minute over design concerns.

The main argument for doing nothing is the concern over how to integrate it in a forwards-compatible and backwards-compatible way. This proposal thus looks to resolve those ambiguities in the Luau grammar so as to avoid this pitfall.

## Drawbacks

### Use of `in` keyword as infix operator

By allowing `in` at the start of a statement, we preclude the use of `in` as an infix operator at any point in the future. There have been some discussions about a similar operator in the past, but they have not seen any clear support, so this proposal decided to use this keyword.

### Roblox - Property casing
Today in Roblox, every index doubly works with camel case, such as `part.position` being equivalent to `part.Position`. This use is considered deprecated and frowned upon. However, even with variable renaming, this becomes significantly more appealing. For example, it is common you will only want a few pieces of information from a `RaycastResult`, so you might be tempted to write:

```lua
local position = Workspace:Raycast(etc)[]
```

...which would work as you expect, but rely on this deprecated style.
