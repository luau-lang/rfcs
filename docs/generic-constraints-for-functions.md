# Constraints on generic parameters for functions

## Summary

Implement `G: T` syntax to specify that a function with generic parameter `G` has restricted the domain of that generic to be a subtype of `T`.

Implement "`with`" clauses to specify that a type function must be reducible given a set of generic parameters as arguments.

Implement inference rules to automatically infer constraints on generics based on the body of the function:

```luau
local function f(s)
    print(string.sub(s, 1, 2))
    return { s }
end
```

The inferred type of `f` should be `f: <T: string>(s: T) -> { T }`. Today, it is inferred as `f: (s: string) -> { string }` ,
preventing the use of string singletons.

```luau
local function f(s)
    print(s .. ", world!")
    return s
end
```

`f` should be inferred `f: <T with concat<T, string>>(T) -> T`. Today, it raises a type error. A constraint on the generic
is required in this case to allow it to be checked.

## Motivation

Programmers write code that supports many types, but not all.

Many programmers write code without annotations. We should still be able to infer good types for this code, to the best of our ability.
Inference sometimes needs more tools to be able to express such code.

## Design

At callsites, we will have two types of constraints on generics to check:

* Subtype constraints: The instantiated type must be a subtype of a given type
* Witness constraints: The instantiated type(s) are arguments to a type function that must then be reducible

If a constraint is violated, we raise a type error at the callsite.

### Explicit constraints

Subtype constraints may be specified on individual generic parameters with `:`:

```luau
local function iOnlyTakeStringsAndStringSingletons<T: string>(s: T): reallyReallyCoolTypeFunction<T>
    -- ...
end
```

Witness constraints may be specified after all arguments to the type function with `where`:

```luau
local function f<T, U where check<T, U>, UnrelatedV>(t: T, u: U, v: V)
end
```

### Inferred constraints

We currently fail to generalize a free type with a mixed polarity if it has upper bounds. We can instead now convert those upper bounds
into a subtype constraint on the generic:

```luau
-- Currently, `x` is `string`. It should be generic as it is both an input and output of the function,
-- but because there is an upper bound, we are just taking that instead for soundness.
-- This leads to us losing singletons.
local function f(x)
    print(string.len(x))
    return x
end

-- With the new inference rules, we instead infer `f: <T: string>(x: T) -> T`, maintaining our ability to
-- keep singletons without implicitly upcasting them to their `string` primitive.
```

Additionally, if we find an internal type function that also depends on the generic, we take it as a witness constraint on the generics
it depends on.

Furthermore, if we find an internal type function that depends on a free type without a mixed polarity, it must be generalized to allow:

```luau
local function printConcat(s)
    print(s .. ", World!")
end
```

, to be checked.

## Prior art

* C#
* TypeScript

## Drawbacks

More complexity.

We may infer meaningless generics for functions like `<T: number>(x: T) -> T`. There are no "strict" subtypes of `number`, as of the time of writing.

## Alternatives

Do not.

Luau currently supports intersecting generics with other types in the function parameter position, and will raise a type error if the intersection
is uninhabited at callsites. However, this is much less rich than the features proposed in this RFC.

Use a different keyword for witness constraints, such as `where` or `needs`.
