# Trailing comma support in calls and function definitions

## Summary

Luau will support trailing commas in the following places:

```lua
local function definition(
    x: number, -- <-- Here, previously a syntax error
)

call(
    a,
    b, -- <-- Here, previously a syntax error
)

type Fn = (
    x: number,
) -> ()
```

## Motivation

Currently, trailing commas are supported in several places in Luau.

```lua
local t = {
    a,
    b,
    c,
}

-- Which even means...
f {
    a,
    b,
    c,
}

type Type = {
    a: number,
    b: number,
}
```

However, they are not supported in function definitions, types, or calls. This is an unintuitive syntax error and it is obvious what the user intended to do.

## Design

We will make these adjustments to the grammar:

```patch
funcargs =
-   '(' [explist] ')'
+   '(' {exp ','} [','] ')'
    | tableconstructor
    | STRING
```

This allows `f(a, b, c,)`.

There is no difference between `f(a(),)` and `f(a())`. Meaning, if `a()` returns `1, 2, 3`, then `f(a(),)` will be equivalent to `f(1, 2, 3)`. This is consistent with tables and `{ a(), }`.

```patch
parlist =
-    bindinglist [',' '...']
+    (bindinglist [',' | ',' '...']
    | '...' [':' (Type | GenericTypePack)]
```

This results in:

```lua
function f(
    a,
    b, -- New!
) end

function f(
    a,
    ... -- The usual
) end

function f(
    a,
    ..., -- NOT allowed
)
```

Trailing comma is not allowed after `...` as it is never correct to put more arguments after the ellipsis. This is consistent with:
- JavaScript - Supports `function f(a, b,)`, but `function f(...a,) {}` results in "Rest parameter must be last formal parameter".

Though this is not universally agreed upon:
- Rust's unstable C variadics support `unsafe extern "C" fn f(a: u32, ...,) {}`, though this syntax is not finalized.
- PHP supports `function f($a, ...$args,) {}`
- Go *requires* a trailing comma after every argument, including variadics, if split over multiple lines.

```patch
BoundTypeList =
    [NAME ':'] Type
        [',' BoundTypeList]
+       [',']
    | '...' Type
```

This results in:

```lua
type Fn = (x: T,) -> () -- Allowed, new!
type Fn = (x: T, ...U) -> () -- Allowed
type Fn = (x: T, ...U,) -> () -- Not allowed
```

## Drawbacks

There are no interesting drawbacks.

## Alternatives

Supporting `function f(...,)` is easily doable if the rules seems unintuitive.
