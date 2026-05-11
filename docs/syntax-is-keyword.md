# `is` keyword

## Summary

Add a new expression of the form: `<expr> is not? <typename>`, describes what
`typename` means, the name resolution logic for `typename`, and how it relates
to type refinements.

## Motivation

Luau has a lot of different ways to do type refinements depending on what kind
of test you need to do.

The `is` keyword was originally designed to check whether a given object is an
instance of a class, and no more than that. This RFC proposes that there is a
generalization that allows it to work with any arbitrary primitives, and can
even be extended to work with host-defined types.

This means it unifies many different (but not all) type refinement patterns into
one single expression, and as a side benefit, can be partially evaluated at
compile time.

These can be superseded by `is`:

- `type(x) == "userdata"`,
- `typeof(x) == "Instance"`,
- `x:IsA("Part")` or some function that performs a type test in a particular
  environment, and now
- `class.isinstance(obj, Class)`.

These will _not_ be superseded by `is`, because they don't have a `typename`
identity that could be reified into the VM, and that's ok.

- `option.type == "some"`, and
- `is_some(option)`.

And this works without requiring you to write the logical conjunctions upfront.
Currently type refinements requires you to spell it out the long way, as opposed
to the second function:

```luau
function is_part_old(x: unknown): boolean
  return typeof(x) == "Instance" and x:IsA("Part")
end

function is_part_new(x: unknown): boolean
  return x is Part
end
```

## Design

### Syntax

Before we get to the EBNF for the `is` keyword, we need to make a few things
clearer. From the original EBNF we have:

```ebnf
exp ::= asexp { binop exp } | unop exp { binop exp }
asexp ::= simpleexp [`::` Type]
```

Firstly, the `asexp` production rule is overloaded to be simultaneously
`simpleexp` _and_ a type ascription, because ``[`::` Type]`` is optional. If we
make that not optional, then `asexp` is the production rule for only `asexp` and
nothing else, and we add the `simpleexp { binop exp }` back in `exp`.

Secondly, we rename `asexp` to `ascriptionexp`.

```diff
- exp ::= asexp { binop exp } | unop exp { binop exp }
+ exp ::= simpleexp { binop exp }
+       | unop exp { binop exp }
+       | ascriptionexp { binop exp }

- asexp ::= simpleexp [`::` Type]
+ ascriptionexp ::= simpleexp `::` Type
```

Now we can notice that `ascriptionexp` does not consume `unop`. Leaving aside
the precedence opinions of `ascriptionexp` as out of scope of this RFC. We want
the `is` expression to bind less tightly than unary operators but more tightly
than binary operators, and be exclusive to `ascriptionexp`.

The reason we need to get this right is because of `-v is Vector3`. If we got it
wrong, `-v is Vector3` would be parsed as `-(v is Vector3)` instead of `(-v) is
Vector3`, as is currently the case with type ascription.

The EBNF for this expression is very small.

```diff
- exp ::= simpleexp { binop exp }
-       | unop exp { binop exp }
-       | ascriptionexp { binop exp }
+ exp ::= simpleexp { binop exp }
+       | unaryexp { binop exp }
+       | isexp { binop exp }
+       | ascriptionexp { binop exp }

+ unaryexp ::= unop exp

+ complexexp ::= unop complexexp
+              | simpleexp

+ isexp ::= complexexp `is` [`not`] typename
+ typename ::= `nil`
+            | `function`
+            | NAME {`.` NAME}
```

Now with this, the `complexexp` consumes as many `not`, `#`, and unary `-` as
part of the subexpression on the left of `is`. This makes the `is` keyword bind
less tightly than unary operators, and binds more tightly than binary operators,
sharing no common production rules with `ascriptionexp`. So `not x is boolean`
is `(not x) is boolean`, ditto `-v is Vector3` is `(-v) is Vector3` and so on.

### Built-in `typename`s

In a barebone environment, the default set of typenames are all the following
built-in Luau VM primitives:

- `boolean`,
- `buffer`,
- `function`,
- `integer`,
- `nil`,
- `number`,
- `string`,
- `table`,
- `thread`,
- `userdata`,
- `vector`,
- `object`, and
- `class`.

### Host-defined `typename`s

A host with its own environment is allowed to register additional `typename`s to
the typename registry, under the following constraints:

1. It must not overwrite any built-in `typename`s.
2. It cannot overwrite any `typename`s that have already been registered.
3. No registered `typename`s can be retracted from the registry.
4. The `typename` registry lives in the `global_State`, so no module-specific
   host-defined `typename`s exist.

The expectation is that `typename`s are globally stable and consistent, and no
`typename`s can be invalidated at any arbitrary point in time. If the host
environment has two different types of the same name, that's a design issue and
the responsibility does not rest with us. Qualified paths are available as a
disambiguation mechanism.

### Value namespace vs typename namespace

One of the suggestion in the classes RFC was to allow `class.isinstance` to work
with certain primitives that have a built-in global library, e.g.

```luau
function is_string(x: unknown)
  return class.isinstance(x, string)
end
```

But consider what happens if you want to check if it's `table` or `userdata` or
`object`:

```luau
function is_shapelike(x: unknown): boolean
  return class.isinstance(x, table)
      or class.isinstance(x, ???) -- no userdata library
      or class.isinstance(x, ???) -- no object library
end
```

This RFC replaces that idea altogether by generalizing it to work with `nil`,
`function`, `boolean`, `number`, `userdata`, and any possible host-defined
`typename`s, as well as any future VM primitives that come up with no global
library associated. Not to mention that `coroutine` library creates a value
called `thread`, which is a name mismatch. That _technically_ works from the
operational semantics point of view, but when you try to apply type system logic
to that, it catastrophically falls apart in a rapid fashion.

For that reason alone, the `typename` on the right of `is`/`is not` does not
have the usual name resolution logic that ordinary identifiers have. This is
crucial as it allows various primitive types and host-defined `typename`s to be
testable without a real value to rely on.

Ordinarily, names are resolved through the "value namespace," but as evident by
the fact that no value exists for certain primitives, `typename`s have to live
in a different namespace, called the "typename namespace."

The value namespace is the union of the local scope and the global scope,
whereas the typename namespace is the union of the local scope and the typename
registry, never interacting with the global scope.

### Name resolution of `typename`

Whether the typename resolves to the local scope or the typename registry
depends on the _root_ name of the qualified path to the typename. The root name
is simply the first `NAME` in the grammar ``NAME {`.` NAME}``.

1. If the root name is an `AstExprLocal`, then it is treated as an ordinary
   expression resolving through `__index` and all that. For instance, the
   qualified typename path `M.A` is an ordinary expression `M.A` that requires
   runtime to resolve the field `A` dynamically.

   ```luau
   const M = require("./mod")

   function f(x: unknown): boolean
     return x is M.A
   end
   ```

   This is equivalent to the following:

   ```luau
   const M = require("./mod")

   function f(x: unknown): boolean
     const C = M.A
     return x is C
   end
   ```

2. If the root name is `AstExprGlobal`, then it is treated as a name lookup in a
   `typename` registry. The qualified typename path is decomposed as a list of
   strings on the stack before calling the resolver in the typename registry to
   return a predicate function. There's a partial evaluation opportunity here as
   an [optimization](#partial-evaluation).

At no point does it go through the global scope. This gives us an opportunity to
avoid the monkeypatching issues and allows primitives to be testable through the
`is` keyword without any global library for them.

As an example, this one goes through the typename registry because `boolean` is
not a local, so the compiler generates code that puts `"boolean"` on the stack,
and dispatches the typename registry resolver to resolve to a predicate function
that determines whether the given value is a boolean.

```luau
function is_boolean(x: unknown): boolean
  return x is boolean
end
```

Similarly, this one goes through the typename registry because `Enum` is not a
local, so `"Enum", "LuauTypeCheckMode"` is on the stack, and again dispatches
the typename registry resolver. In Roblox's environment, this resolves to
another predicate function that determines whether the enum item is a member of
the `LuauTypeCheckMode` enumeration.

```luau
function is_luau_typecheck_mode(x: unknown): boolean
  return x is Enum.LuauTypeCheckMode
end
```

### Negation of `is`

The `not` keyword is allowed to come after the `is` keyword for ergonomic
reasons, but also to disambiguate between these two possible parse trees:

1. `(not b) is boolean`, and
2. `not (b is boolean)`.

This way, we get to define the expression `not b is boolean` to be the first,
and the expression `b is not boolean` to be the second.

### Calling conventions

There are two kinds of functions at play here:

1. Predicate functions, and
2. `typename` registry resolver.

The predicate function has the type:

```luau
type Pred = (L: lua_State, x: unknown, polarity: boolean) -> boolean
```

The `polarity: boolean` parameter is purely for the host to have an opportunity
to coax the C/C++ compiler to generate a short-circuiting logic for when
`polarity == false` in the case that the host has authored a predicate as a
series of logical conjunctions. This `polarity` is load-bearing, and the host is
required to write a lawful predicate function satisfying:

```
pred(L, x, true) == not pred(L, x, false)
```

This is trivial by writing `polarity == (x and y)`.

The typename registry resolver has the type:

```luau
type Resolver = (L: lua_State, ...: string) -> Pred
```

Once the `Resolver` returns a `Pred`, that given qualified typename path can be
cached at the top-level module without needing to be resolved over and over
again, as per the [partial evaluation optimization](#partial-evaluation).

If the user has written a qualified typename which is not found in the registry,
then the resolver returns a predicate function that always throws an error. This
matches the behavior of `x is MyMod.MyNonexistentClass` which only throws an
error if the control flow enters through this expression.

### Optimization ideas

This is just to illustrate a few ideas. The analysis and design is left as an
exercise for the VM maintainers. These optimizations presuppose the constraints
that are required in this RFC.

#### Partial evaluation

Suppose I have a function that iterates over a list of instances, and only
collect the ones that are `BasePart`s:

```luau
local function collect_base_parts(xs: {Instance})
  local result = {}

  for _, x in xs do
    if x:IsA("BasePart") then
      table.insert(result, x)
    end
  end

  return result
end
```

Currently, this would need to perform a `NAMECALL`, checks if `x` is a
`userdata`, is host-owned (which unlocks `__namecall` and `__type`), then
finally invokes the `__namecall`. In there, it then needs to know what `__type`
it is, and what string `"BasePart"` even means, before it finally dispatches the
real predicate function that checks if `x` is a subclass of `BasePart`, which is
nontrivial because an instance that is a subclass of `Instance` but not
`BasePart` has to know the common superclass to realize that any additional
checks are an exercise in futility.

```luau
local function collect_base_parts(xs: {Instance})
  local result = {}

  for _, x in xs do
    if x is BasePart then
      table.insert(result, x)
    end
  end

  return result
end
```

In this version, `x is BasePart` can be partially evaluated as `IS_BASE_PART(x)`
where it is only waiting for a single value. It doesn't even care about
`__namecall`, `userdata`, `__type`, or what `"BasePart"` means, because all that
work has been done beforehand via the `typename` registry. So the above becomes:

```luau
const IS_BASE_PART = --[[ C function ]]

local function collect_base_parts(xs: {Instance})
  local result = {}

  for _, x in xs do
    if IS_BASE_PART(x) then
      table.insert(result, x)
    end
  end

  return result
end
```

#### Prefix-sharing

When registering a typename, the host could also declare that it requires a set
of predicates to also be true. If all of these prerequisites are true, then and
only then does the VM actually invoke the predicate function with baked-in
assumptions of all prerequisites.

For example, for `Part`, it requires `FormFactorPart`, which requires
`BasePart`, which requires `PVInstance`, which requires `Instance`, which
requires `Object`, which requires `userdata`, then if we know the predicate
function for `Part` is not true, there are still a prefix of predicates that
have had to execute and do not necessarily need to be executed again, e.g. `x is
Part or x is Folder`.

#### Decision tree

The compiler could also generate bytecode that fuses `IS_PART` and `IS_FOLDER`
into one single predicate function and the runtime can then traverse the
registry and compute an optimal ordering of predicates to fire first and return
a jump target.

Obviously finding the "optimal ordering of predicates" is NP-hard, so
[Maranget-style heuristics][maranget] is required if performance becomes a
problem with prefix-sharing.

[maranget]: https://dl.acm.org/doi/epdf/10.1145/1411304.1411311

### Grammar limitations

You cannot use parentheses in the right side of `is`/`is not`.

```luau
local is_boolean = b is (boolean)
```

This is already parsed as two distinct statements:

```luau
local is_boolean = b
is(boolean)
```

This is intentional. It is almost assured that in real world code, people will
want to write the typename on the right of `is` with a name as the first token,
so we're taking advantage of that grammar, even if it's strictly less flexible
as a grammar, e.g. Python allows `b is (bool if b else bool)`.

We don't expect anyone to want to write `x is not if b then A else B`. In the
unlikely event that someone did, the clearer form `if b then x is not A else x
is not B` is available.

You cannot mix `::` and `is` without parentheses. The associativity of `::` and
`is` is confusing, so they are mutually exclusive and requires parentheses.

Consider `x is M.A :: T` for some arbitrary `T`. Is this:

1. `x is (M.A :: typeof(MyClass))`, or
2. `(x is M.A) :: boolean`?

Obviously the first example cannot be parsed (as per the first grammar
limitation), but recall that if the root name is a local, then it's plausible
that `M.A :: typeof(MyClass)` is valid as one interpretation of the above
expression. But trying to cast the typename is by definition nonsensical if the
root name is a global, since in effect, you're trying to cast that typename to
some other class type when it is already statically known.

Not to mention that you literally have `MyClass` available to you already. Just
write `x is MyClass`.

The second parse is pointless since `is` always has type `boolean`, unless we
prove `x <: T`, then its type is `true`, and dually `x </: T` => `false`, but if
that were so, we can raise a lint warning that this check is redundant.

Also consider `x :: a is number`. If Luau decides to implement user-defined type
guards and the syntax for that is `x is number`, then the expression is not
backward compatible with `x :: <type> is <typename>` due to the ambiguous parse.
This is probably fine and not a problem, but it's better to be conservative
here.

### A different lens on the `typename` namespace

Note that this is purely pedagogical. The compiler does not literally have this
exact same operational model, i.e. no thunks are materialized, no `setfenv`
calls are made, none of that.

This construct is equivalent to the combination of things that Lua/Luau
programmers already understand, if they know how `setfenv` works, they can
internalize why the `typename`s are in scope only on the right side of `is` and
not in scope as an ordinary expression.

One way to internalize the intuition of the `typename` namespace is to treat the
`is` expression as a macro.

```luau
x is <typename> -> is(x, function() return <typename> end)
```

By extracting the `typename` into a thunk and then treating it as an ordinary
expression, we can then pass the thunk into the `is` function, and the `is`
function is simply defined as:

```luau
const function is(x: unknown, t: () -> (class | (unknown) -> boolean)): boolean
  -- If `pred` is a `function`, that can only be from the typename registry, so
  -- the error message only reports an error if the user had some expression
  -- that did not evaluate to a `class`.

  local f = setfenv(t, typename_registry)
  local pred = f()
  return if typeof(pred) == "function" then pred(x)
    else if typeof(pred) == "class" then class.instanceof(x, pred)
    else error(`expected a \`class\`, got \`{typeof(pred)}\``)
end
```

Now it's immediately obvious to us that the built-in global scope is completely
inaccessible to the thunk, and the `typename_registry` is now set as the global
scope in the thunk. The `typename_registry` just looks like this in the barebone
environment:

```luau
const typename_registry = {
  boolean = function(x) return type(x) == "boolean" end,
  buffer = function(x) return type(x) == "buffer" end,
  ["function"] = function(x) return type(x) == "function" end,
  integer = function(x) return type(x) == "integer" end,
  ["nil"] = function(x) return type(x) == "nil" end,
  number = function(x) return type(x) == "number" end,
  string = function(x) return type(x) == "string" end,
  table = function(x) return type(x) == "table" end,
  thread = function(x) return type(x) == "thread" end,
  userdata = function(x) return type(x) == "userdata" end,
  vector = function(x) return type(x) == "vector" end,
  object = function(x) return type(x) == "object" end,
  class = function(x) return type(x) == "class" end,
}
```

And then the host-defined typenames are able to extend this registry:

```luau
const typename_registry = {
  ... everything as before ...,

  -- Roblox env
  Instance = function(x) return typeof(x) == "Instance" end,
  Part = function(x) return typeof(x) == "Instance" and x:IsA("Part") end,
  Folder = function(x) return typeof(x) == "Instance" and x:IsA("Folder") end,

  -- Enumerations
  Enum = {
    LuauTypeCheckMode = function(x)
      return typeof(x) == "EnumItem" and x:IsA("LuauTypeCheckMode")
    end,
  },
}
```

Now, `x is boolean` becomes `is(x, function() return boolean end)` under this
lens, and that resolves to `typename_registry["boolean"]`, which then returns
`function(x) return type(x) == "boolean" end`, and likewise `x is MyClass`
becomes `is(x, function() return MyClass end)`, which simply delegates to
`class.isinstance(x, MyClass)`.

## Drawbacks

This requires teaching programmers to not blindly treat the thing on the right
of `is` as an expression.

Any `typename` typos are silent until the control flow reaches through the `is`
expression, which then throws an error. This is already a problem with existing
type guards anyhow, e.g. `typeof(x) == "nill"` will silently do nothing and
always returns `false` (unless by chance its `__type` is `nill`...). Dynamically
typed programming languages are already full of this class of bugs, e.g. field
projections, mistyped locals resolves to a global, etc. The type system can be
used to rescue users from typos, but the status quo remain no worse than before.

This also requires the host to populate the `typename` registry to participate
in the `is` keyword with all possible types from their environment. A solution
that could alleviate this pain is to provide a hook for when the `typename` is
not found in the registry, so that populating the registry can be done on-demand
and keep the startup time and memory cost as small as possible. Nevertheless,
this is one more thing that the host now has to do _if_ they want to cooperate
with the `is` keyword.

## Alternatives

1. Instead of using `is`, we could use `instanceof` keyword. But that makes it
   sound like it only works for `x instanceof C` for some `typeof(x) <: object`
   and `typeof(C) <: class`, and you lose out on a few other generalizations
   that this RFC enables, e.g. the arbitrary host-defined type guards.

2. Instead of adding the `typename` namespace, we let the expression on the
   right of `is` resolve through the global namespace. This loses the
   unification opportunity wrt the type guards story.

3. Instead of returning a predicate function that always throws an error if the
   user has written a `typename` which is not found in the registry, have the
   registry resolver throw that error immediately.

   This wasn't chosen because that would become observable under partial
   evaluation. If the compiler lifts `x is NotFound`, that would cause the
   module to fail to initialize, as opposed to matching the behavior of `x is
   SomeLocal` where `SomeLocal` is `nil` or some non-class which throws an error
   only when the expression `x is SomeLocal` is being executed.

4. Instead of giving the `polarity` to the predicate functions, the host always
   write a predicate that assumes `polarity == true` and the VM negates the
   result on their behalf. This removes the `polarity` parameter from the
   calling convention.
