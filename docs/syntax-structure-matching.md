# Structure matching

## Summary

Agree on the generic syntax for *structure matching* - a prerequisite to implementing destructuring in any part of Luau.

This is intended as a spiritual successor to the older ["Key destructuring" RFC by Kampfkarren](https://github.com/luau-lang/rfcs/pull/24), which was very popular but requires more rigour and wider consensus to have confidence implementing the feature.

**This is not an implementation RFC.**

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

This proposal does not specify any specific locations where this syntax should appear. Instead, the aim is to get consensus on the syntax we would be most comfortable with for all instances of destructuring we may choose to implement at a later date.

In particular, this proposal punts on implementation at sites of usage:

- Destructuring re-assignment (as opposed to destructuring `local` declarations)
- Defaults for destructured fields (unclear how this interacts with function default arguments)
- Unnamed function parameters (destructuring a parameter doesn't name the parameter)
- Type declarations on keys (for providing types when destructuring a function argument)

The purpose of this proposal is to instead find consensus on specific syntax for the matching itself.

This proposal puts forward a superset of syntax, able to match any table shape, with logical and simple desugaring, giving a rigorous foundation to the previously agreed-upon concise syntaxes.

### Structure matching

This proposal will use the term *structure matcher* to refer to syntax for retrieving values from table structures.

The most basic structure matcher is a set of empty braces. All matching syntax occurs between these braces.

```Lua
local { } = data
```

Empty structure matchers like these are not invalid (they still fit the pattern), but aren't very useful - linting for these makes sense.

String keys can be specified between the braces to introduce them as members of the namespace.

```Lua
local { foo, bar } = data
```

This desugars to:

```Lua
local foo, bar = data.foo, data.bar
```

If the member should be renamed, a new contextual `as` keyword can be used, with the key on the left and the member name on the right.

```Lua
local { foo as red, bar as blue } = data
```

This desugars to:

```Lua
local red, blue = data.foo, data.bar
```

This proposal does not preclude supporting arrays, nested matching or non-string keys down the line (see Alternatives for proposed syntaxes), but it takes the opinion that these considerations do not outweigh the need for an internally consistent and ergonomic base variant for the most common case of matching string keys in maps.

## Alternatives

#### Symbol for renaming

Using a symbol like `=` or `:` for renaming was considered, but was rejected because it was ambiguous whether the left hand side was the key, or the right hand side. By contrast, a keyword like `as` naturally implies an order.



#### Dot keys 

A previous version of this proposal considered using dots at the start of identifiers. The original motive was to disambiguate 
visually between arrays and maps (especially for people familiar with other languages). 

This was ultimately rejected after much deliberation as Luau already has a different pattern for working with arrays, and so would be unlikely to extend this syntax to work with them.

Additionally, it didn't seem valuable enough to contort our syntax to follow the conventions of other languages; it was instead preferred to prioritise internal consistency of Luau and ergonomics.

```Lua
{ .foo, .bar }
```

#### Nested structure

A previous version of this proposal considered matching nested structure, but this was rejected as it wasn't clear what the use case would be.

```Lua
{ .foo.bar }
```

### Unpack syntax

A dedicated array/tuple unpacking syntax was considered, but rejected in favour of basic syntax.

```Lua
{ unpack foo, bar }
```

Instead, for unpacking arrays, this proposal suggests:

```Lua
foo, bar = table.unpack(data)
```

### Indexing assignment

A large amount of effort was poured into finding a way of moving the destructuring syntax into the middle of the assignment.

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

This proposal aims to tackle such design concerns in stages, agreeing on each step with open communication and space for appraising details.

## Drawbacks

### Structure matchers at line starts

This design precludes the use of a structure matcher at the start of a new line, among other places, because of ambiguity with function call syntax:

```Lua
local foo = bar

{ } -- bar { }?
```

Such call sites will need a reserved keyword prior to the matcher to dispel the ambiguity.

```Lua
in { myFoo = .foo }
```

That said, this would only manifest if attempting to add destructuring to re-assignment operations, which isn't done in other languages and has limited utility (only really being useful for late initialisation of variables).
