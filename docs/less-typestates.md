# Use local initializers as the upper bound

## Summary

If a local has an initializer that is not `nil`, use it as the upper bound for
that local, otherwise give it the `unknown` type. Proposes that local shadowing
should be made idiomatic and to disable all `LocalShadow` lints by default.

## Motivation

> NB: This is setting up the motivation with the historical context in mind.

When Luau's new type solver was being developed, typestates did not exist, and
we generated a fresh metavariable for each unannotated binding in a `local`
declaration. This turned out to be problematic because locals are not
parameters of a function and thus don't follow the same typing rules:

1. Each use of the local binding is negatively used, reducing the allowed domain
   the local binding can range over. This means the upper bound will _always_
   approach `never`, even if the program is valid!
2. Each assignment of the local binding is positively used, which is an
   over-approximation because it affects all _uses_ of the local binding, after
   _and_ before the assignment.

If your system produces multiple incorrect solutions for a given problem, it
usually means the system is not granular enough to capture the nuances, which
implies your system needs to grow more complexity. In this case, it's pretty
clear the metavariable is not the correct tool to use here.

```luau
export type ok<a> = { type: "ok", value: a }
export type err<e> = { type: "err", error: e }
export type t<a, e> =
  | ok<a>
  | err<e>

local Result = {}

function Result.ok<a>(value: a): ok<a>
  return { type = "ok", value = value }
end

function Result.err<e>(error: e): err<e>
  return { type = "err", error = error }
end

function Result.foldr<a, e>(
  xs: {t<a, e>},
  init: a,
  f: (acc: a, value: a) -> a
): t<a, e>
  local res = Result.ok(init)

  for _, x in xs do
    if x.type == "ok" then
      res.value = f(res.value, x.value)
    elseif x.type == "err" then
      -- semantically equivalent to `return x`
      -- but intentionally written this way to demonstrate a problem
      res = x
      break
    else
      local absurd: never = x
      error("absurd")
    end
  end

  return res
end
```

Under the metavariable approach in the `Result.foldr` function, the statement
`local res = Result.ok(init)` would infer `ok<a> <: 'res <: unknown`. At `res =
x`, we also add `t<a, e>` to the lower bound of `'res`, giving us `t<a, e> <:
'res <: unknown`. And finally, in the statement `return res`, we add `t<a, e>`
to the upper bound of `'res`, giving us the final type `t<a, e> <: 'res <: t<a,
e>`.

If we replaced `'res` by the upper bound, then an error would be raised at
`res.value = f(res.value, x.value)` since `res` is `ok<a> | err<e>`, which is
nonsensical. The same is true when replaced by its lower bound.

Therefore the metavariable approach is incorrect, since it cannot be replaced
by the upper bound nor the lower bound in a way that allows the type system to
decide whether the program is well-typed. Typestates was the solution to this
problem: instead of introducing a fresh metavariable, any unannotated locals
such as `local x = 5` is defaulted to the equivalent of `local x: unknown = 5`,
and the _next use_ of `x` would infer `number`, rather than `unknown`.

Similarly, in this smaller contrived program, if we replaced `x` by its lower
bound, then `x: number | string` and all uses of `x` would be ill-typed.

```luau
local x = 5
something_that_takes_only_numbers(x)
x = "five"
something_else_with_only_strings(x)
```

As mentioned before, to correct this flaw we would need to grow some complexity
in order to allow the program to type check. Typestates solves it, so we just
implemented it and made the decision that `local x = 5` would stand for `local
x: unknown = 5`.

As it turns out, people don't like this. People seem to want the initializer of
the `local` to be inferred as the upper bound instead of `unknown`. This can be
problematic in certain cases like if the initializer was simply `nil`, e.g.
`local x` or `local x = nil` or `local x = id(nil)`, which prevents you from
assigning a non-nil value to `x`. This, along with the consistency that no
locals can ever range over anything smaller than `unknown`, are precisely why we
decided to default to `unknown` for unannotated locals.

## Design

Instead of defaulting unannotated locals to the upper bound of `unknown`, we
will instead use the type of the initializer as the upper bound for every local
bindings, e.g. `local x = 5` gives us `local x: number = 5`, and `local y = "y"`
gives us `local y: string = "y"` (or `y: "y"` if `y` is constrained by a
singleton).

```luau
local x = 5 -- x: number
x = "five" -- type error

local y1 = "y" -- y1: "y"
local y2: "y" = y1 -- fine
```

The exception to this rule is if the initializer is missing or `nil`, in which
case the type of the local is `unknown`. It's not terribly useful for a local to
only range over `nil`.

```luau
local late_bound = nil
late_bound = "five" -- fine
late_bound = 5 -- fine
```

The reason why this seems to make sense is because there's nothing here that we
could use to _pin_ the local to a specific type in such cases, because of all
sorts of nontrivial programs such as:

```luau
local result

-- replace this branch with _any possible program_ that initializes `result`
if math.random() > 0.5 then
  result = 7
elseif f(x) and g(y) then
  result = "something else"
end

-- result: number | string | nil
```

If we tried to use a fresh metavariable to infer an upper bound, it would be a
futile exercise, and would quickly run into terrible UX: for each time a local
is negatively used with different types, the upper bound approaches `never`,
resulting in all assignments to the local to be ill-typed. A ridiculous notion
because the problem isn't coming from the assignments themselves.

## Drawbacks

Lua 5.1 decoupled the use of locals from the registers they represent. This is
good because locals are virtual registers, but Lua 5.1 still kept the 200 locals
per function limit. Luau inherited this design choice (and increased the number
of physical registers from 200 to 256, the limit of `uint8_t`).

This means locals are not free from a register allocation point of view, due to
finite number of virtual registers. There are several ways this can be fixed:

1. Reuse the same physical register for any locals that have been shadowed, and
   do not count such locals towards the limit.
2. Use liveness analysis: count the number of simultaneously live registers
   across all program points in a function. Throw a compile error if the number
   of registers exceed the maximum number of physical registers, then remove the
   200 locals per function limit.

Option #1 is obviously braindead easy to do. Option #2 is harder to do, but
would liberate the users from the maximum number of locals per function since it
is extremely unlikely for the user to have exceeded 256 physical registers just
from locals (unless used as function arguments, but you cannot have more than
256 values passed on the stack, so this is a non-issue).

On top of that, there is a possible optimization coming in the future where
tables becomes a scalar and its fields are locals until they escape. This will
make the locals and scalar tables compete for the same finite resource of
physical registers, which actually motivates liveness analysis in order to allow
more shadowing, and allow even more tables to be scalars.

Note that this does not propose increasing the number of physical registers,
only the number of virtual registers.

## Alternatives

Do nothing. The argument that "every type system throws an error when assigning
a different type to a local" is weak given that Luau is fundamentally a
dynamically typed programming language, and languages that _grew_ with a type
system have the privilege to make this an error whereas Luau might not. This has
the consequence of having to support pre-existing idioms.

Introduce a `let` syntax that disallows assignment _of a different type_ at type
checking time, e.g. `let x = 5; x = "five"` compiles to the same bytecode, but
is ill-typed. `local`s still retain its current behavior.
