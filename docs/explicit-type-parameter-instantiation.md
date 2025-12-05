# Explicit type parameter instantiation

**Status**: Implemented

## Summary

We will add support for the ability to bind a function defined with generics to specific types at the time of expression. This behavior is analagous to C++'s `f<T>` and Rust's `f::<T>`, where `f` is defined as a function accepting a type parameter `T`.

## Motivation

There are several cases where Luau either makes it overly difficult to declare a type in a function call or overly verbose. For example, let's consider something akin to React's bindings:

```lua
local function createBinding<T>(value: T): React.Binding<T>
    -- code
end
```

If we wanted a binding to a number, then Luau can trivially infer this type:

```lua
-- 1999 is a number, so moneyBinding is React.Binding<T>
local moneyBinding = React.createBinding(1999)
```

In other cases, the type is more complex. For example, if we want money to be `number?`, then we cannot simply pass in `nil`, as that will infer as `React.Binding<nil>`. Furthermore, while there are theoretically cases like these where Luau can expand out a `nil` into `number?`, it is still beneficial to want this to be restricted at the type to prevent confusing debugging or potential for the type to infer wider than intended.

There are two ways to do this. One is to declare the type of `moneyBinding` itself to `React.Binding`. This would look like...

```lua
local moneyBinding: React.Binding<number?> = React.createBinding(nil)
```

This is not only quite verbose, but requires that the user know the exact type of the return value of `React.createBinding`, *and* requires that it really be as trivial as just passing in the types. For example, `React.Binding` could be an even more complex type that asks for some sort of initial state type, or setter type, and so on. In other cases, it might even be defined as an inline table, or as a non-exported type.

The other alternative is something like:

```lua
local moneyBinding = React.createBinding(nil :: number?)
```

However, `::` has the ability to cast things in unsound manners. Both these calls are accepted with `::` when we would not want it in this case:

```lua
local function f(x: number?)
    React.createBinding(x :: number) -- Allowed
end

type Object = { a: number }
React.createBinding({} :: Object) -- Allowed
```

So while it would work in some cases, it provides an extra bit of uncertainty to the correctness of the call.

In other cases, type parameters are used in places where there is no immediate value binding to that type. For example, a Signal type might be defined as:

```lua
type Signal<Args...> = {
    fire: (Args...) -> (),
    connect: ((Args...) -> ()) -> (),
}

local function createSignal<Args...>(): Signal<Args...>
```

The usual local assignment bindings work fine here:

```lua
local playerTookDamageSignal: Signal<Player, number> = createSignal()
```

...but only if this binding is actually declared through a local. In a case like:

```lua
Signals.playerTookDamageSignal = createSignal()
```

...we either have to put this out into another variable above, which increases surface area, or use `::`, which has the problems described previously. This is not a problem specific to function calls, but is one of the places where they would be useful.

## Design

We propose allowing the following:

```lua
-- Saving `f<<T>>` as a variable to called later
local a = f<<T>>

-- Parameterized function call as statement
f<<T, U>>()

-- Parameterized function call as expression
local b = f<<T, U>>()

-- Calling through index
local c = t.f<<T>>()
local d = t:f<<T>>()
local e = t["f"]<<T>>()
```

For parsing, we will do this by extending `var` to allow `prefixexp '<<' TypeList '>>'`. This will bind the prefixexp equivalent to the substitution of the type parameters. That is to say...

```lua
local function f<T, U>() end

-- Binds `T` and `U` to `number` and `string`
local f1 = f<<number, string>>

-- Binds just `T` to number. The unspecified `U` type is anonymous in the same way it would be if this were just `f`.
local f2 = f<<number>>
```

When this new syntax is used on a value that is not a function, or does not take those types, a static-time error will be given. This syntax has zero runtime effect.

### Interoperability with metatables

There are three problems that arise when attempting to instantiate a table with a metatable attached, namely:

1. The forall quantifier is nested under the metamethod for that table's metatable, giving three levels of indirection.
2. It's unclear what metamethod to instantiate without a way to disambiguate, since a metatable can have multiple polymorphic metamethods.
3. If the `__metatable` field is present, `getmetatable` will throw an error, making it unsafe to work around this with `getmetatable(t).__call<<T>>`.

Hence, we must produce a type error if the term is not a function type of the correct kind, full stop.

If it was supported, explicit type instantiation through the metatable would require disambiguation using the surrounding syntactic context. That is, `t<<number>>.x` instantiates the `__index` metamethod, or `t<<string>>(...)` instantiates the `__call` metamethod, and so forth. The tradeoff there is that the type inference engine will now have an extra `std::optional<Metamethod>` parameter that is only used when checking an expression of the form `e<<T>>`.

This trivial (and contrived) example shows what it would look like in practice, and generalization to more complex cases is straightforward, but would obscure the core point.

```luau
type T = setmetatable<{ x: number, y: string }, {
  __index: <K>(T, K) -> rawget<T, K>,
  __call: <K>(T, K, rawget<T, K>) -> rawget<T, K>
}>

local function f(t: T)
  local x = t<<"x">>.x -- instantiates `__index` with `K = "x"`.
  local y = t<<"y">>("x", "hello!") -- instantiates `__call` with `K = "y"` (and a type error `"x" </: "y"`)
end
```

## Drawbacks

The syntax proposed in this RFC would close the door on the ability to add a bitwise left-shift operator to Luau. This is because this would make the following usage ambiguous:

```lua
-- Assuming we have right-shift, because there's no way we add one without the other
return f<<a, b>>(c)
```

This is either a parameterized function call of `f(c)` with `a` and `b` as types, or a return of two values--`f << a`, and `b >> (c)`.

Because there would be no ability to add bitwise shifts to the language, it is unlikely that any bitwise operators would be added, as it would be a stark omission.

With that said, the ability to perform these kinds of shifts is already available through the `bit32` library, and [Luau's compatibility page](https://luau.org/compatibility#lua-53) has the following words on the topic in reference to their availability in Lua 5.3:

> If integers are taken out of the equation, bitwise operators make less sense, as integers aren’t a first class feature; additionally, bit32 library is more fully featured (includes commonly used operations such as rotates and arithmetic shift; bit extraction/replacement is also more readable). Adding operators along with metamethods for all of them increases complexity, which means this feature isn’t worth it on the balance.

The syntax is also unfamiliar not just in the context of Luau but in the context of programming languages in general--`<<` and `>>` as type parameter specification is completely unique to this call alone.

## Alternatives

### Syntax

First, what we absolutely cannot pick:

- `f<T>()` is the choice of C++, but is ambiguous with `f<a, b>(c)` in the same way we mentioned with bitwise operators above. C++ only disambiguates this with an extra pass using type information that would dramatically inflate the complexity of all Luau parsers.
- `f::<T>()`, named the "turbofish", is the choice of Rust. This is not strictly ambiguous, but it would require infinite lookahead as it conflicts with casting generic functions. In `local x = f::<T>(() -> T) -> ()`, which is valid code today, we cannot know this is a type assignment until we get to the first `)`, as until that point we might have code that looks like `local x = f::<T>((x))`. This is not a cost we would like to add to the parser.

For unambiguous alternatives, `f.<T>` and `f:<T>` are among the most discussed. This would look like:

```lua
local moneyBinding = React.createBinding.<number>()

-- or...
local moneyBinding = React.createBinding:<number>()
```

The downside of these is that they blur the lines between runtime and static, in the sense that `React.createBinding.` starts out as a runtime concept, followed by the purely static `<number>`. As for `:`, it carries the baggage of `x:y()` carrying the interesting behavior of adding `self` to the call, which it does not do here.

There is also not necessarily a reason that we have to provide symmetrical operators, so something like `f!T, U()` is reasonably parseable, but is not obviously better and is even more unlike any other language's syntax.

Ultimately, this RFC chooses `<<T>>` as a least worst option that is fluid to type, easy to read, and easy to remember.

### Limiting to function calls, instead of expressions

This is allowed at the expression level as it allows for the obvious use of `f<<T>>()`, but with the ability to be more expressive and pass it into places like:

```lua
local function f<A, B, C>(
    a: (A) -> (),
    b: (B) -> (),
    c: (C) -> ()
) end

f(f1, f2<<number>>, f3)
```

The equivalent syntax is allowed in both C++ and Rust, but it is a rare use case that could add significant complexity. In most cases, keeping it to function calls will work just fine.

### Not doing it

As always, we can not do this. It is worth noting that in the original generic functions RFC, parameterized function calls are explicitly omitted in a [section entitled "Turbofish", linked here](https://rfcs.luau.org/generic-functions.html#turbofish).

Relevant to our purposes is the following section:

> Some languages don’t have a way to specify the types at call site either, Swift being a prominent example. Thus it’s not a given we need this feature in Luau.

This is still the case for Swift at time of writing.

