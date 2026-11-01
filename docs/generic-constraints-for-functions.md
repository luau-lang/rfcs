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
local function counter<T>(c: T)
    return function(x: T)
        c += x
        return c, x
    end
end
```

The inner function should be inferred as `<with add<T, T>>(T) -> (T, T)`. Today, it raises a type error. A constraint on the generic
representing that `T + T` is valid is required in this case to allow it to be checked.

## Motivation

Programmers write code that supports many types, but not all.

Many programmers write code without annotations. We should still be able to infer good types for this code, to the best of our ability.
Inference sometimes needs more tools to be able to express such code.

## Design

At callsites, we will have two types of constraints on generics to check:

* Subtype constraints: The instantiated type must be a subtype of a given type
* inhabitance constraints: The instantiated type(s) are arguments to a type function that must be inhabited

If a constraint is violated, we raise a type error at the callsite.

### Explicit constraints

Subtype constraints may be specified on individual generic parameters with `:`:

```luau
local function iOnlyTakeStringsAndStringSingletons<T: string>(s: T): reallyReallyCoolTypeFunction<T>
    -- ...
end
```

Inhabitance constraints may be specified with the `with` keyword:

```luau
-- GenericTypeList :: NAME ['with' SimpleType] [',' GenericTypeList] | 'with' SimpleType | GenericTypePackParameter {',' GenericTypePackParameter}
local function f<T, U with check<T, U>, UnrelatedV>(t: T, u: U, v: V)
end
```

They may appear anywhere in the generic type parameter list, but not after a generic type pack as type functions do not currently accept type packs.
If multiple inhabitance constraints are present, they are separated with commas and each require an individual `with` keyword.

```luau
-- Ok!
type f = <A, B with check1<A, B>>(A, B) -> ()
-- Ok! `T` is a generic of an enclosing function.
type g = <with check2<T>>(T) -> ()
type h = <A with check1<A>, check1<T>>(A, T) -> ()
-- They appear at the end of the generic type list, after generic packs.
type i = <T, U..., with checkPack<(U...) -> ()>(T, U...) -> ()
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

Additionally, if we find an internal type function that also depends on the generic, we take it as a inhabitance constraint on the generics
it depends on.

```luau
local function counter<T>(c: T)
    return function(x: T)
        c += x
        return c, x
    end
end
```

The `+=` operation mints an `add<T, T>` type function. This is to be copied into a inhabitance constraint, representing that `T` must support
`+` with itself, but this is only relevant if the *inner* function is called. Thus, the constraint is applied onto that inner
function instead of the `counter` function itself.

## Subtyping

Intrinsically, function types which accept more kinds of arguments are subtypes of function types which accept less kinds of arguments.

```luau
<T>(T) -> T <: <T: string>(T) -> T
<T>(T) -> T <: <T with add<T, number>>(T) -> T
```

Additionally, we know from:

```luau
(unknown) -> unknown </: <T>(T) -> T
```

, the relation is held:

```luau
(string) -> string </: <T: string>(T) -> T
```

## Prior art

* C#
* TypeScript

## Drawbacks

More complexity.

We may infer meaningless generics for functions like `<T: number>(x: T) -> T`. There are currently no strong subtypes of `number`.

We introduce new syntax, which can make visually parsing text of the language more difficult.

We are further overloading the meaning of `:`. It would now have both the meaning of method calls, binding annotations, as well as generic
subtyping constraints.

We introduce a new keyword, `with`, that does not appear to have precedence in prior art. C# uses `where`, and TypeScript uses `extends`.

## Alternatives

Do not.

Luau currently supports intersecting generics with other types in the function parameter position, and will raise a type error if the intersection
is uninhabited at callsites. However, this is much less rich than the features proposed in this RFC.

Use a different keyword for inhabitance constraints, such as `where` or `needs`.
