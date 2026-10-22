# Constraints on generic parameters for functions

## Summary

Implement `G: T` syntax to specify that a function with generic parameter `G` has restricted the domain of that generic to be a subtype of `T`.

Implement inference rules to automatically infer constraints on generics based on the body of the function:

```luau
local function f(s)
    print(string.sub(s, 1, 2))
    return { s }
end
```

The inferred type of `f` should be `f: <T: string>(s: T): { T }`. Today, it is inferred as `f: (s: string): { string }` ,
preventing the use of string singletons.

Furthermore, we also want to infer "witness" types:

```luau
local function f(s)
    print(s .. ", world!")
end
```

`f` should be inferred `f: <T, U: concat<T, string>>(T) -> ()`. Today, it raises a type error. A constraint on the generic
is required in this case to allow it to be checked.

## Motivation

Programmers write code that supports many types, but not all.

## Design

### Explicit constraints

Constraints may be specified on individual generic parameters with `:`:

```luau
local function iOnlyTakeStringsAndStringSingletons<T: string>(s: T): reallyReallyCoolTypeFunction<T>
    -- ...
end
```

### Inferred constraints

We currently fail to generalize a free type a mixed polarity if it has upper bounds. We can instead now convert those upper bounds
into a constraint on the generic.

Additionally, if we find an internal type function that also depends on the generic, we place it as a witness in the generic parameter list
and ensure it reduces successfully at callsites.

## Prior art

* C#
* TypeScript

## Drawbacks

More complexity.

We may infer meaningless generics for functions like `<T: number>(x: T) -> T`. There are no "strict" subtypes of `number`, as of the time of writing.

## Alternatives

Do not.

Luau currently supports intersecting generics with other types in the function parameter position, and will raise a type error if the intersection
is uninhabited at callsites, however, this is much less rich than the feature proposed in this RFC.
