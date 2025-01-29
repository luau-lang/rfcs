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

The purpose of this proposal is to instead find consensus on specific syntax for the matching itself.

This proposal puts forward a superset of syntax, able to match any table shape, with logical and simple desugaring, giving a rigorous foundation to the previously agreed-upon concise syntaxes.

### Structure matching

This proposal will use the term *structure matcher* to refer to syntax for retrieving values from table structures.

The most basic structure matcher is a set of empty braces. All matching syntax occurs between these braces.

```Lua
{ }
```

Empty structure matchers like these are not invalid (they still fit the pattern), but aren't very useful - linting for these makes sense.

#### Basic matching

This is the most verbose, but compatible way of matching values.

Keys are specified in square brackets, and are allowed to evaluate to any currently valid key (i.e. not `nil`, plus any other constraints in the current context).

To save the value at that key, an `=` is used, and an identifier is specified on the right hand side, showing where the value will be saved to.

```Lua
{ [1] = foo, [#data] = bar }
```

This desugars to:

```Lua
foo, bar = data["foo"], data[#data]
```

#### Dot keys with names

Keys that are valid Luau identifiers can be expressed as `.key` instead of `["key"]`.

```Lua
{ .foo = myFoo, .bar = myBar }
```

This desugars once to:

```Lua
{ ["foo"] = myFoo, ["bar"] = myBar }
```

Then desugars again to:

```
myFoo, myBar = data["foo"], data["bar"]
```

#### Dot keys without names

When using dot keys, the second identifier can be skipped if the destination uses the same identifier as the key.

```Lua
{ .foo, .bar }
```

This desugars once to:

```Lua
{ .foo = foo, .bar = bar }
```

Then desugars twice to:

```Lua
{ ["foo"] = foo, ["bar"] = bar }
```

Then desugars again to:

```
foo, bar = data["foo"], data["bar"]
```

#### Unpacking

Instead of listing out consecutive numeric keys, `unpack` can be used at the start of a matcher to implicitly key all subsequent items. This is useful for arrays and tuple-style tables.

```Lua
{ unpack foo, bar }
```

This desugars once to:

```Lua
{ [1] = foo, [2] = bar }
```

Then desugars again to:

```Lua
foo, bar = data[1], data[2]
```

`unpack` skips dot keys and explicitly written keys:

```Lua
{ unpack foo, [10] = bar, baz, .garb }
```

This desugars once to:

```Lua
{ [1] = foo, [10] = bar, [2] = baz, ["garb"] = garb }
```

Then desugars again to:

```Lua
foo, bar, baz, garb = data[1], data[10], data[2], data["garb"]
```

It is invalid to specify an identifer without a key if `unpack` is not specified, for disambiguity with other languages.

#### Nested structure

A structure matcher can be specified instead of an identifier, to match nested structure inside of that key. This is compatible with consecutive keys and dot keys.

No `=` is used, as this is not an assigning operation.

*Open question: should we? or perhaps a different delimiter for visiting without binding? Discuss in comments.*

```Lua
{{ .foo { .bar = myBar } }}
```

This desugars once to:

```Lua
{ [1] { ["foo"] { ["bar"] = myBar } } }
```

Then desugars again to:

```Lua
local myBar = data[1]["foo"]["bar"]
```

## Alternatives

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

Such call sites will need a starting token (perhaps a reserved or contextual keyword) to dispel the ambiguity.

We could mandate a reserved or contextual keyword before all structure matchers:

```Lua
match { .foo = myFoo }
in { .foo = myFoo }
```

But this proposal punts on the issue, as this is most relevant for only certain implementations of matching, and so is considered external to the main syntax. We are free to decide on this later, once we know what the syntax looks like inside of the braces, should we agree that braces are desirable in any case.

### Matching nested structure with identifiers

In *Nested structure*:

> An identifier and a structure matcher cannot be used at the same time. Exclusively one or the other may be on the right hand side.

This is because allowing this would introduce ambiguity with dot keys without names:

To illustrate: suppose we allow the following combination of nested structure and dot keys with names:

```Lua
{ .foo = myFoo { .bar } }
```

Which would desugar to:

```Lua
local myFoo, bar = data.foo, data.foo.bar
```

If we switch to dot keys without names:

```Lua
{ .foo { .bar } }
```

How would this desugar?

```Lua
local foo, bar = data.foo, data.foo.bar
-- or
local bar = data.foo.bar
```

This is why it is explicitly disallowed.

### Consecutive key misreading

Consider this syntax.

```Lua
{ foo, bar, baz }
```

This desugars to:

```Lua
{ [1] = foo, [2] = bar, [3] = baz }
```

But an untrained observer may interpret it as:

```Lua
{ .foo = foo, .bar = bar, .baz = baz }
```

Of course, we have rigorously defined dot keys without names to allow for this use case:

```Lua
{ .foo, .bar, .baz }
```

But, while it fits into the desugaring logic, it is an open question whether we feel this is sufficient distinction.

One case in favour of this proposal is that Luau already uses similar syntax for array literals:

```Lua
local myArray = { foo, bar, baz }
```

But one case against is that JavaScript uses brackets/braces to dinstinguish arrays and maps, and a Luau array looks like a JS map:

```JS
let { foo, bar, baz } = data;
```

Whether this downside is actually significant enough should be discussed in comments though.

Consecutive keys are arguably most useful when used with tuple-like types like `{1, "foo", true}`, as they can match each value by position:

```Luau
{ id, text, isNeat }
```

However, Luau does not allow these types to be expressed at the moment. It isn't out of the question that we could support this in the future, so the door should likely be left open for tuple-like tables.