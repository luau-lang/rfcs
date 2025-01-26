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

The `[]` indexing operator is extended to support explicitly reading from multiple (potentially) non-consecutive keys at once, by provided a comma-separated set of expressions.

```Lua
local numbers = {3, 5, 11}

local three, five, eleven = numbers[1, 2, 3]
```

This will desugar to this behaviour exactly:

```Lua
local numbers = {3, 5, 11}
local three, five, eleven = numbers[1], numbers[2], numbers[3]
```

Arrays can read out an inclusive range of values from consecutively numbered keys. This is specified with `[x : y]`, where `x` and `y` evaluate to a number.

```Lua
local numbers = {3, 5, 11}

local three, five, eleven = numbers[1 : #numbers]
```

Negative numbers are allowed, with symmetric behaviour to Luau standard library functions that accept negative indices (offset from the end of the table).

```Lua
local numbers = {3, 5, 11}

local three, five, eleven = numbers[1 : -1]
```

Multiple index expressions run the risk of duplicating information already specified by the names of identifiers in declarations or re-assignments.

```Lua
local nicknames = { amelia = "amy", bethany = "beth", caroline = "carol" }

local amelia, bethany, caroline = nicknames["amelia", "bethany", "caroline"]

amelia, bethany, caroline = nicknames["amelia", "bethany", "caroline"]
```

If the expression appears on the right hand side of assignment to explicit identifiers, the keys may be implied to be the identifier strings.

```Lua
local nicknames = { amelia = "amy", bethany = "beth", caroline = "carol" }

local amelia, bethany, caroline = nicknames[]

amelia, bethany, caroline = nicknames[]
```

Implied keys do not correlate to identifier position; when positional indices are desired, use range shorthand instead.

## Alternatives

### Braces around identifier list during assignment

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

### Open ranges

A syntax for indexing open ranges was considered, where the start and/or end of the range would be implied to be the first/last index.

```Lua
-- closed range
local three, five, eleven = numbers[1 : -1]

-- range with open end index
local three, five, eleven = numbers[1 :]

-- range with open start index
local three, five, eleven = numbers[: -1]

-- range wih open start & end indexes
local three, five, eleven = numbers[:]
```

This is plausible to do, and is supported by ranges in other modern programming languages:

``` Rust
// closed range
let slice = numbers[1..3]

// range with open end index
let slice = numbers[1..]

// range with open start index
let slice = numbers[..3]

// range in open start & end indexes
let slice = numbers[..]
```

This proposal does not push for open ranges so as to limit the scope of the proposal, but they are explicitly left open as a option should we wish to pursue them at a later time.

### Alternate range delimiters

This proposal selected `:` as the token to act as the range delimiter.

The `..` token was considered. However, this was discarded because it would be ambiguous with string concatenation.

```Lua
local foo = bar["hello" .. "world"] -- is this correct?
```

The `...` token was considered. However, this was discarded because it would be ambiguous with a variadic multiple index.

```Lua
local function foo(...)
    return bar[...] -- open range or variadic?
end
```

The `in` token was considered. However, this was discarded because it may appear misleading, sound awkward, or be incompatible with future expression syntax using `in` as an infix operator.

```Lua
-- every third index? 
-- a 1/3 chance? 
-- test if 1 is found in 3?
local three, five, eleven = numbers[1 in 3] 
```

A prefix `in` was also considered, but was decided against since it could be mistaken for non-range indexing.

```Lua
local three, five, eleven = numbers[in 1, 3] -- in indexes 1 and 3?
```

Combining `in` with another one of the tokens was considered, but it was considered too verbose to be desirable.

```Lua
local foo = bar[in 1 ... 3]
```

`in` also has the disadvantage of being associated with `for`, where it does not function as a range delimiter, nor a multiple indexer:

```Lua
for key, value in numbers do -- `in` has a different job here
```

### Don't add ranges / Don't add implicit keys

This proposal explicitly intends to add two shorthands:
- Range shorthand is designed for arrays.
- Implicit key shorthand is designed for maps.

This ensures both are equally easy to destructure, and that the syntax explicitly looks different for both to avoid confusion.

One of the previous grounds for RFC rejection was that the suggested destructuring method was asymmetric and unclear between dictionaries and arrays, and so this proposal intends to avoid that.

### Alternate implicit key syntax

An alternate `[local]` syntax was considered for implicit keys, to make it more visible after a function call:

```Lua
local useState = require("@react")[local]
```

There is nothing functionally wrong with this, so the proposal doesn't discard this possibility, but it was deemed to have a slightly odd shape compared to other Luau constructs. It could also have an adverse effect on line length.

### Implicit key renaming

Renaming identifiers was considered with implicit keys, and should be possible with full backwards compatibility, and support for arbitrary expressions.

```Lua
amy in "amelia", beth in "bethany", carol in "caroline" = nicknames[]
```

Notably, this would allow the use of implicit keys with only some keys renamed, useful for avoiding namespace collisions:

```Lua
local useEffect, useReactState in "useState", useMemo = require("@react")[]
local useState = require("@game/reactUtils")[]
```

However, it means we would need to reject this syntax for expressions without implicit keys:

```Lua
local foo in "bar" = 2 + 2 -- not valid
```

However, renaming is already *technically* doable via shadowing:

```Lua
local useState = require("@react")[]
local useReactState = useState
local useState = require("@game/reactUtils")[]
```

It is also possible with the explicit syntax:

```Lua
local useEffect, useReactState, useMemo = require("@react")["useEffect", "useState", "useMemo"]
local useState = require("@game/reactUtils")[]
```

This proposal doesn't *reject* the idea of renaming, but considers it out of scope for now as it could potentially introduce a significant level of added complexity, and can be worked around in many ways.

### Implicit positional keys

It was considered whether to add a second "positional" implicit mode, and disambiguate between positional and identifier modes using syntax.

```Lua
-- Hypothetical syntax
amelia, bethany, caroline = nicknames[.]
three, five, eleven = numbers[#]
```

However, all syntax choices led to suboptimal outcomes, where an open range would be equally concise, equal in functionality, and more consistent with the rest of the proposal. Not to mention, all syntax that is added is syntax that could block future RFCs.

As such, the proposal is fine leaving implicit keys with only one mode for simplicity, as it is wildly counterproductive to have two equally valid ways of doing the same thing.

```Lua
-- As proposed (w/ open ranges)
amelia, bethany, caroline = nicknames[]
three, five, eleven = numbers[:]

-- As proposed (closed ranges only)
amelia, bethany, caroline = nicknames[]
three, five, eleven = numbers[1 : -1]
```

### Don't do anything

This is always an option, given how much faff there has been trying to get a feature like this into Luau!

However, it's clear there is widespread and loud demand for something like this, given the response to the previous RFC, and the disappointment after it was discarded at the last minute over design concerns.

The main argument for doing nothing is the concern over how to integrate it in a forwards-compatible and backwards-compatible way, which arises from previous RFCs focusing solely on the assignment list which is notably sensitive to ambiguity issues.

This proposal thus looks elsewhere in the Luau grammar, and finds other places where ambiguity is not omnipresent, so as to avoid this pitfall.

## Drawbacks

### Array/map distinction

A common sticking point in previous destructuring designs has been how to disambiguate array destructuring from map destructuring.

This proposal attempts to solve this functionally by introducing two syntaxes, one for each:

```Lua
-- As proposed (w/ open ranges)
amelia, bethany, caroline = nicknames[]
three, five, eleven = numbers[:]

-- As proposed (closed ranges only)
amelia, bethany, caroline = nicknames[]
three, five, eleven = numbers[1 : -1]
```

While not strictly as intuitive as the design of previous RFCs, it solves every hard design problem nicely. That said, the proposal is still open to evolving this syntax based on feedback if the distinction is deemed unclear.

### Roblox - Property casing
Today in Roblox, every index doubly works with camel case, such as `part.position` being equivalent to `part.Position`. This use is considered deprecated and frowned upon. However, even with variable renaming, this becomes significantly more appealing. For example, it is common you will only want a few pieces of information from a `RaycastResult`, so you might be tempted to write:

```lua
local position = Workspace:Raycast(etc)[]
```

...which would work as you expect, but rely on this deprecated style.
