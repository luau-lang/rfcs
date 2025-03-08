# Frozen table syntax

## Summary

Add a syntax to table literals to allow them to be frozen from construction.

```Lua
local struct = {:
    name = "Bob",
    age = 24
:}

print(struct.name) --> Bob
struct.name = "Joe" --> attempt to modify a readonly table
```

## Motivation

Luau currently supports "freezing" a table to make it read-only and shallowly immutable.

```Lua
local data = { foo = 2, bar = 4 }

data.foo = 2 -- ok
data.baz = 5 -- ok

table.freeze(data)

data.foo = 15 -- not ok
data.baz = nil -- not ok
data.garb = 20 -- not ok
```

Frozen tables are beneficial in a wide number of cases, due to a few concrete benefits;

- Once a table is frozen, the type becomes shallowly "set in stone" - all top-level (i.e. non-table) key-value pairs are exactly known and will never change
- Frozen tables prevent interior mutability, making local reasoning more reliable, improving coroutine safety, and allowing library authors to make stronger assumptions
- Code writers can use frozen tables to protect against mistakes introduced while writing new code

Due to their various technical and UX benefits, frozen tables have grown to become the preferred way of expressing immutability in Luau.
Community projects such as [Fusion](https://github.com/dphfox/Fusion) actively recommend their use for all new code, due to their more predictable behaviour unlocking aggressive optimisations that would not otherwise be possible (see [Fusion's documentation on frozen table comparisons](https://elttob.uk/Fusion/0.3/tutorials/best-practices/optimisation/)).

As a result of this, one emerging common pattern is to exploit Lua 5.1's function call syntax sugar to create "frozen table literals":

```Lua
local struct = table.freeze {
    name = "Bob",
    age = 24
}
```

The use of function call syntax sugar is generally discouraged in modern Luau, and is often seen as confusing to less experienced users, as function calls can masquerades as custom language-level features.
Generally, a program like StyLua must be employed to remove these occurrences.

However, when written to modern Luau standards, the entire statement grows noisier:

```Lua
local struct = table.freeze({
    name = "Bob",
    age = 24
})
```

This is amplified when used in the context of nested function calls or nested structures, which is common when working with libraries that provide immutable table utilties:

```Lua
TableUtils.join(
    table.freeze({
        within = container,
        alignment = table.freeze({ x = 0.5, y = 0.5 })
    }),
    table.freeze({
        group_bounds = table.freeze({ min = vector.create(0, 0, 0), max = vector.create(1, 1, 1) })
    })
)
```

This leaves the feature in an awkward place, where it provides great theoretical benefits, but is impractical to express without using undesirable language features.
Ideally, frozen tables should have only marginal cost to adopt, to reduce friction and encourage wider usage. This inexpressibility is a serious cost.

## Design

For conciseness, this proposal allows the contents of a table literal to be wrapped in colons `:` to freeze it.

```Lua
local empty = {::}
local struct = {:
    name = "Bob",
    age = 24
:}
```

This is unambiguous and exactly equivalent to:

```Lua
local empty = table.freeze({})
local struct = table.freeze({
    name = "Bob",
    age = 24
})
```

In more complex code, this syntax approaches the readability of non-frozen tables:

```Lua
TableUtils.join(
    {:
        within = container,
        alignment = {: x = 0.5, y = 0.5 :}
    :},
    {:
        group_bounds = {: min = vector.create(0, 0, 0), max = vector.create(1, 1, 1) :}
    :}
)
```

The syntax is unambiguous and compatible with both arrays and mixed tables.

```Lua
local fibbonaci = {: 1, 1, 2, 3, 5 :}

local mixed = {:
    red = "shoe",
    blue = "clue",
    green = "hue",
    1, 2, 3
:}
```

Colons were selected as the option least likely to introduce future ambiguity or forward incompatibility.
Colons are currently only valid in method calls `foo:bar()`, type annotations `foo:bar`, and type casts `foo::bar`; in all cases, these cannot appear at the start of a table literal.

By introducing a "frozen table literal", the (shallow) shape of the table is guaranteed to never change from construction.
Furthermore, if only frozen table literals are present inside of a definition, it's trivial to guarantee the deep shape of the table from construction too.
This is particularly useful for optimisation, where the allocation of the table can be shared or skipped (though this can affect referential identity).
A similar optimisation is already used for some anonymous function definitions.

In particular, this should allow Luau to skip all extra allocations when returning tables from functions, a sticking point in previous discussions with power users (see [Named function type returns](https://github.com/luau-lang/rfcs/pull/67) for relevant discussion).

```Lua
-- With the optimisation mentioned above, these two functions would have almost identical performance.
-- This is because the frozen table does not have to be allocated with each function call.

local function foo()
    return "Bob", 24
end

local function foo()
    return {:
        name = "Bob",
        age = 24
    :}
end
```

Of less importance (but still relevant); this syntax is very easy to write, due to the proximity of `:` to `{` on ISO and ANSI keyboards, as well as both characters using the same physical Shift modifier key.

## Drawbacks

`table.freeze` exists today, so this proposal does not add any new concepts to Luau's mental model. It is purely syntax sugar.

While syntax sugar is good as it does not increase the level of irreducible complexity in Luau, it is bad because it still increases implementation complexity, and requires some level of education in all cases.

Other than that, there aren't any other discernible downsides.

## Alternatives

### Use something other than colons

A range of punctuation was tested. 

#### Bar

`{| |}` was considered, but it could lead to parsing difficulty if frozen-ness was added as a property of tables in the type system.

```Lua
type Problematic = {| "foo" -- from this context alone, can you tell if this is a frozen table?

-- The two possible completions are *technically* unambiguous but only with the full context:
type Problematic = {| "foo"}
type Problematic = {| "foo" |}
```

#### Equals

`{= =}` was considered, but is awkward to write and can visually interfere with key-value pairs.

```Lua
-- this can be hard to visually interpret at a glance due to the repeated use of =
local foo = {= bar = "baz" =}
```

#### Plus

`{+ +}` was considered; while it is easier to write, it would block us from adding a unary plus operation in the future.

```Lua
local foo = {+1.0 -- from this context alone, can you tell if this is a frozen table?

-- The two possible completions are *technically* unambiguous but only with the full context:
local foo = {+1.0}
local foo = {+1.0+}
```

#### Star

`{* *}` was considered, but is slightly less ergonomic to write and more visually prominent. Other than that, there are no discernible benefits or drawbacks.

### Don't do anything

As always, we could elect not to implement this syntax and instead direct people to use `table.freeze`.

This is undesirable as it would make immutable-style code much harder to write, requiring a fully-qualified function call for every frozen table.
Developers would have to find other ways of reducing the burden, such as using a non-standard single-letter alias like `local f = table.freeze`.
These kinds of aliases are already controversial (e.g. `local e = React.createElement`) as many are concerned that it deteriorates the readability of code.
Ideally, Luau should be designed in a way where such workarounds are avoided by ensuring the langauge itself is ergonomic to use by default.

That said, there is a cost to every feature implemented, and each addition has the potential to whittle away some of Luau's essential simplicity.
This proposal is not overly complex - in fact, it's almost reductively simple - but no proposal has zero cost. Even simple features, on mass, can bloat a language.
Against that backdrop, this proposal justifies consideration by poiting to the pervasive utility of frozen tables in modern Luau code, and the significant technical advantages of encouraging their use.