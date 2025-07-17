# Table Destructuring Syntax



## Summary
Introduce new syntax for unpacking values from tables.

```lua
local &{ foo, ["@bar"] = bar } = thing
-- foo == thing.foo
-- bar == thing["@bar"]
```

This RFC addresses issues brought up from previous table destructuring RFC's.



## Motivation
Aliasing table items through variable assignments is a common pattern in luau, especially when working with large libraries. However doing this with a lot of items becomes repetitve:
```lua
local useEffect = React.useEffect
local useMemo = React.useMemo
local useState = React.useState
```

With destructuring syntax this could be shortened to the following:
```lua
local &{ useEffect, useMemo, useState } = React
```

## Design

We propose that the table destructuring syntax is prefixed with an ampersand (`&`) symbol. Without a prefix destructuring becomes ambiguous in certain scenarios, such as the one below.
```lua
baz = foo
{ bar } = test
```
*(This is ambiguous as it can be interpretted as `baz = foo { bar }; = test` or `baz = foo; { bar } = test`).*

### Key destructuring
We propose the following syntax for destructuring by keys:
```lua
&{ ["foo"] = foo } = thing
-- foo == thing.foo
```

If the key can be expressed as a valid variable name then a shorthand can be used instead:
```lua
&{ foo } = thing
```

Shorthands can be destructured with a different name:
```lua
&{ foo = bar } = thing
-- bar == thing.foo
```

We propose the following syntax for nested key destructuring:
```lua
&{ ["@foo"] = &{ bar } } = thing
-- bar == thing["@foo"].bar
```
This also works with shorthands.

### Array Destructuring
We propose the following syntax for destructuring arrays:
```lua
&{{ one, two, three }} = thing
-- one == thing[1]
-- two == thing[2]
-- three == thing[3]
```

You can omit a certain index via the `nil` keyword:
```lua
&{{ one, nil, three }} = thing
-- one == thing[1]
-- three == thing[3]
```

We propose the following syntax for nested array destructuring:
```lua
&{{ one, &{{ apple }}, three }} = thing
-- one == thing[1]
-- apple == thing[2][1]
-- three == thing[3]
```

### Mixed Destructuring
Key and array destructuring can be used together.

```lua
&{
    ["@foo"] = foo,
    { one, nil, three },
} = thing
-- foo == thing["@foo"]
-- one == thing[1]
-- three == thing[3]
```

### Local Destructuring
We propose that if a destructure assignment is preceeded with `local` then all of the destructured variables will be local.
```lua
local &{ hello, world } = thing
```

### Function Argument Destructuring
We propose that function arguments can be destructured:

```lua
type Props = {
    apple: any,
    pear: any
}

function test(&{ apple, pear }: Props)
    ...
end
```
*(The type annotation is not neccesary, its purely there for demonstration purposes).*

### Type Destructuring.
We propose special behaviour for destructing types from requires:
```lua
&{ type Foo } = require(...)
```

You can also precede a destructure assignment with `type`. This is particularly helpful for destructuring multiple types without having to repeat the `type` keyword, although it does have the caveat of only allowing type destructuring.
```lua
type &{ Foo, Bar, Baz } = require(...)
```

Types can also be destructured with a different name:
```lua
&{ type Foo = Biz } = require(...)
-- type Biz == type require(...).Foo
```


## Drawbacks
Prefixing table destructuring with a symbol may not fit into the language as historically luau has generally not used symbol prefixes. However there are some examples of symbol prefixes (especially in recent years):
- length operator (`#thing`).
- function attributes (`@native`).
- negation operator (`~SomeType`).



## Alternatives
Alternative ideas for table destructuring have been proposed before, most of which have been rejected:
- https://github.com/luau-lang/rfcs/pull/24 (`local {.a, .b} = t`)
- https://github.com/luau-lang/luau/pull/629 (`local { a, b } = t`)
- https://github.com/luau-lang/rfcs/pull/95 (`local { a, b } = t`)

The main reasons for their rejection include ambiguity issues, not handling arrays, and not handling non-local assignments.

