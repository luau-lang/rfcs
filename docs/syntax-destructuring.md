# Destructuring

## Summary

Introduce a set of destructuring utilities that:
- work with array & dictionary style tables
- work with both declaration and assignment
- don't require parser backtrack
- are consistent with Luau style

Specifically:

1) A `...` unpacking expression:

```Lua
local array: {string}
local dict: {daisy: string, emerald: string, florence: string}

local amy, becca, chloe = array...
amy, becca, chloe = array...

local daisy, emerald, florence = dict...
daisy, emerald, florence = dict...
```

2) A `...=` unpacking assignment:

```Lua
local array: {string}
local dict: {daisy: string, emerald: string, florence: string}

local amy, becca, chloe ...= array
amy, becca, chloe ...= array

local daisy, emerald, florence ...= dict
daisy, emerald, florence ...= dict
```

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

### Destructuring expression

Today, Luau implements positional unpacking via `table.unpack`, but does not implement keyed unpacking. Additionally, the previous RFC made it clear that many Luau users seek a more concise syntax for this common task.

This proposal suggests the introduction of a new destructuring operator to replace `table.pack` for positional unpacking, and extend it to keyed unpacking.

The `...` variadic token is selected, as it is not currently valid following an expression - it is only valid on its own. It also ties this operator to the concept of variadics and multiple returhs, which is appropriate.

Positional unpacking is simple:

```
local numbers = {3, 5, 11}

local three, five, eleven = numbers...
```



## Alternatives

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

As such, this proposal does not pursue these design directions further.