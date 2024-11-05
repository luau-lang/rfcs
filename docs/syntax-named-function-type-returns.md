# Named function type returns

## Summary

In alignment with [named function type arguments](./syntax-named-function-type-args.md), introduce syntax to describe names of returned values for function types. 

## Motivation

Luau does not include semantic information when describing a function returning a tuple of values. This is especially a problem when they can't be distinguished by type alone:

```Lua
-- are the euler angles returned in order of application, or alphabetical (XYZ) order?
calcEulerAngles: () -> (number, number, number)

-- which one is forearm? which one is upper arm?
solveIK: () -> (CFrame, CFrame)

-- is this quaternion in WXYZ or XYZW format?
toQuat: () -> (number, number, number, number)
```

We previously decided to solve a similar problem with function parameters by introducing named function type arguments, and saw multiple benefits:

- better documentation for uncommented functions
- lower propensity for names to fall out of sync with implementation
- improved comprehension for long and dense type signatures
- a consistent location for external tools to extract identifiers

This was all done with minimal extra design work, and done without introducing new responsibilities to other parts of the language.

```Lua
rotateEulerAngles: (y: number, x: number, z: number) -> ()

applyIK: (upperArm: CFrame, forearm: CFrame) -> ()

fromQuat: (x: number, y: number, z: number, w: number) -> ()
```

Function type returns face almost identical problems today, and stand to gain almost identical benefits.

By providing symmetry with named function type arguments, and introducing names for returns, we can:

- consistently apply the pattern of named list members, reducing the mental overhead of learners and less technical Luau users
- provide a single, well-defined location for placing human-readable identifiers usable by LSP inlay types, autofill, and hover tips
- encourage the colocation of semantic information alongside code to ensure documentation and runtime are updated in concert, and aren't prone to non-local editing mistakes
- improve the legibility of complex type signatures by adding human affordances that would not otherwise be present
- do all this with minimal extension to the language and perfect backwards compatibility

```Lua
calcEulerAngles: () -> (y: number, x: number, z: number)

solveIK: () -> (upperArm: CFrame, forearm: CFrame)

toQuat: () -> (x: number, y: number, z: number, w: number)
```

## Design

This proposal mirrors the existing named function type argument syntax and allows it to be used in return position. This allows returned tuples to be annotated with their contents.

This is added to all locations where return types can be defined:
```Lua
-- function type syntax
x: () -> (return1: number, return2: number)

-- function declaration
local function x(): (return1: number, return2: number)
```

This syntax is fully backwards compatible as only type names, generic names, or type packs are allowed in return position.

Return names are documentative - they are not identifiers that are usable in the function body.

Function type comparisons will ignore the return names, this proposal doesn't change the semantics of the language and how typechecking is performed.

```Lua
-- names of returns are ignored
type Red = () -> (foo: number?, bar: number?)
type Blue = () -> (frob: number?, garb: number?)
local x: Red = something :: Blue
```

Assignments and other expressions will ignore the return names, with the same behaviour and type safety as today. 

However, return names can be interpreted by linters to provide superior linting as a purely additive benefit:

```Lua
local function doStuff(): (red: number, blue: number)
    -- ...
end

local function receiveStuff(blue: number, red: number)
    -- ...
end

-- lint: `doStuff()` returns identically-named values in a different order, change these names to silence
local blue, red = doStuff()

-- lint: `doStuff` sends identically named & typed values to `receiveStuff`, but in the wrong order
receiveStuff(doStuff())
```

## Drawbacks

There is a philosophical disagreement over the purpose of the Luau static type system:
- This proposal was written on the grounds that Luau should have internal syntactic consistency and cross-pollination. Runtime, type checking, and documentation features should enmesh and enrich each other as a unified whole, to ensure everything is kept in sync and is easily comprehensible.
- The primary argument against this proposal is the belief that Luau's type checker should be a "pure proof system", cleanly separated from documentation concerns at every point, so that the type system can exist in a pure realm unconcerned with semantic information.

This feels like a core tension about the direction of type checking features as a whole. We should check in with the community on this.

There is a particular concern that introducing comprehension aids into the type language would imply meaning where there is none. Would users think that return names have functional meaning?

There are other places in Luau where we have previously introduced comprehension aids in the past:

```Lua
-- number literals could have been kept pure, but we chose to add meaningless delimiters for easier comprehension
local ONE_BILLION = 1000000000
local ONE_BILLION = 1_000_000_000

-- generics could have been referenced abstractly, but we chose to give them human-readable names
type Foo<2> = (<1>, <2>) -> <1> | <2>
type Foo<OK, Fallback> = (OK, Fallback) -> OK | Fallback

-- parameters in function types didn't have to have named parameters, but we decided to add them
type Func = (foo: number, bar: number) -> ()
```

In particular, it is established convention that Luau does not rearrange the contents of lists bounded by parentheses, even when list items are named:

```Lua
local function poem(red: number, blue: number)
    print("roses are", red, "violets are", blue)
end

local blue, red = "blue", "red"

poem(blue, red) --> roses are blue violets are red
```

The same principle also applies to multiple assignment in existing parts of Luau:

```Lua
for value, key in pairs({"hello", "world"}) do
    print(value) --> 1 2
end
```

So, this proposal posits that there is already established precedent for such features, and that users understand how comprehension aids function in Luau today.

A common concern is whether these comprehension aids would mislead people into believing that names are significant when considering type compatibility.

However, we already have features that clearly indicate we don't do this.

```Lua
-- names of generics are ignored
type Foo = <A, B>(A) -> B
type Bar = <X, Y>(X) -> Y
local x: Foo = something :: Bar

-- names of parameters are ignored
type Red = (foo: number?, bar: number?) -> number
type Blue = (frob: number?, garb: number?) -> number
local x: Red = something :: Blue
```

So, there is no reason why named function type returns would be any different. 

The remaining questions are non-technical:

- Are Luau types a proof system, or a documentation system, or both? What approach makes sense?
- Is it right for types to be annotated for the comprehension of human readers and writers, or is this excess that doesn't need to be there?
- What does it mean for an identifier to be meaningful? Is it meaningful if linters parse for it, or only if it functionally changes runtime behaviour?

Discussion is invited on these questions - this proposal does not prescribe an answer, but consensus between the community and Luau should be reached, and can help direct future proposals.

## Alternatives

### Add meaning to comments

In response to this proposal, the idea of making comments meaningful was proposed. Instead of being treated as non-functional text, comments would be interpreted using a kind of "documentation syntax". Concrete facts about parameters and return types, including identifiers, would be inferred from them.

This works well for verbose documentation such as explanations or diagrams, which would be inappropriate to try and embed inline. However, there are a number of drawbacks to this approach too.

From discussions with users of Luau, uncommented code is still often written with Luau types for LSP benefits such as good autocomplete and improved linting, especially during accelerated phases of development such as prototyping. Whether these people would adopt comments for better LSP features is an open question - we should be mindful of the friction required to personalise LSP results.

This also raises further non-technical questions:

- should comments be given meaning? are they meaningful if linters parse their contents to provide lints? if a parser skips comments, are they losing non-decorative information?
- what documentation syntax do we use? what rules does it follow? how does it fit into Luau's overall design, if it fits at all?
- what is the scope of documentation syntax? where are the edges of its responsibilities?
- what about previous proposals where we decided against placing features into comments? what do they recommend here?

### Don't do anything

It is technically possible for people to "annotate" returned values today using comments. However, there are multiple very tangible downsides to this.

There is no widely-agreed-upon convention for comment format, and as such, any LSP, linter, or docsite generator looking to adopt standardised rules for interpreting returned names would face an uphill battle.

For a community project to implement support, they would need to discover all of the formats for documentation comments in use by Luau developers, and implement support for all of them by hand. With no official specification to go off, this would invite fragmentation.

There is a very real possibility that not implementing *any* standard way of annotating return type names will discourage people from documenting them at all, reducing the quality of documentation in the Luau ecosystem.

The upshot is that Luau would remain unopinionated and free to ignore comments completely. We punt the problem to the community and all consumers of Luau, who will have to invent their own methods of documenting this info if it is desired.
