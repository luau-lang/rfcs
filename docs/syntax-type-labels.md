# Syntax Type Labels

## Summary

Syntax type labels intended for generic types, allowing users to attach parameter names to types for use in argument types.
This is to improve IntelliSense and autocomplete, and allows custom callbacks to show you argument names instead of just the type.

## Motivation

Currently, you can only provide argument names in a type by doing the following.

```luau
type Callback = (x: number, y: number, z: number) -> ()

local cb: Callback = function()
  --[[
    IntelliSense shows you
		cb: (x: number, y: number, z: number) -> ().
    Luau Playground just shows
		cb: (number, number, number) -> ()
  --]]
end
```

But here in this case, a developer would immediately know what each of these type _represents_. In this case ``x``, ``y`` and ``z``.

Luau and any embedder internally within the implementation are able to define argument names in some way or another to types _procedurally_.
But a developer is not able to replicate everything from the implementation itself.

<br/>

Generic packs exist

```luau
type WrappedSignal<T...> = {
	FireToFoo: (self: WrappedSignal<T...>, targetFoo: Foo, T...) -> (),
}

type MySignal = WrappedSignal<number, string>
```

But in this case, the result of ``MySignal.FireToFoo`` would show up as:
```luau
(WrappedSignal<T...>, Foo, number, string)
-- or
(self: WrappedSignal<T...>, targetFoo: Foo, number, string)
```
without any context or indication on what these provided types ``(number, string)`` are meant to re-present.

But you can see that ``targetFoo: Foo`` is present, but only because function annotations specifically let you describe argument names, which APIs can pick-up on.

<br/>

Luau already allows you to provide parameter names.

```luau
(x: number) -> ()
```

But when generic types are used, this information is no longer available

```luau
type Callback<T...> = (T...) -> ()
```

``T...`` will only emit over types but not labels.

And currently, there isn't even syntax to provide a label when using ``Callback<T...>``



Why are we doing this? What use cases does it support? What is the expected outcome?

## Design

**Syntax Type Labels**, allowing users to provide semantics onto types,
useful for IntelliSense and autocomplete, even for debug tooling within the internals of the language itself.
_(a bit similar to TypeScript)_

Labels are only metadata:
- Labels are NOT enforced in any type checking way
- They don't affect the type itself
- They only improve front-end hints


Type Labels are intended to be used while providing types into generics.
```luau
type Callback<T...> = (T...) -> ()

function Foo(cb: Callback<[targetBar: Bar]> end
-- cb shows (targetBar: Bar) -> ()
```

### Syntax:
- Type Label Syntax is wrapped around these brackets ``[]``.
- They can contain a label through ``[label: type]``, but is optional
- ``[number]`` would just be ``number``
- ``[x: number]`` would just be ``number``, with a label ``"x"``
- Multiple types can appear in it by separating with ``,``
  - ``[x: number, y: number]
  - but this is also fine ``[x: number], [y: number]``

Multiple types can only appear if multiple types are even allowed to be provided.
```luau
type Foo<T...> = (T...) -> ()
type Bar<T> = (T) -> ()

type A = Foo<[a: number, number, label: number]> -- allowed
type B = Foo<[x: number], [y: number]> -- allowed

type C = Bar<[x: number, y: number]> -- Error: More than one type!
```

<br/>

### Usage:
```luau
type WrappedSignal<T...> = {
	FireToFoo: (self: WrappedSignal<T...>, targetFoo: Foo, T...) -> (),
}

type MySignal = WrappedSignal<[amount: number, message: string]>

--[[
  MySignal.FireToFoo would become
    (self: WrappedSignal<T...>, targetFoo: Foo, amount: number, message: string) -> ()
]]
```

Nothing changes about the types own identity, it just gets associated with metadata.

<br/>


In TypeScript you can do
```ts
type Position = [x: number, y: number, z: number]
```
Luau does NOT support typing tuples in this context, so doing the above in Luau, would error.



## Drawbacks

- A new syntax has to be introduced. Another question would be whether "labels" will be the only thing, or whether there will be more.
- Whether syntax should be ``[x: number, y: number, z: number]`` or ``[x: number], [y: number], [z: number]``
- ``[]`` are used by table keys and indexing, or _"arrays"_ in other languages.

## Alternatives

Type functions could expose modifying argument names, but that would only work for function annotations, not on a type alone.
There wouldn't be a way to attach a label to a type.

```luau
type WrappedSignal<T...> = { FireToFoo: (self: WrappedSignal<T...>, targetFoo: Foo, T...) -> (); }
type MySignal = WrappedSignal<(label1: number, label2: string) -> ()>
```
And this would also mean that ``WrappedSignal`` needs to implement a type function now, that takes out the argument names and transforms ``.FireToFoo``.
And that sounds too complicated.

You don't want it to be

You don't want it to be ``( targetFoo: Foo: Player, (label1: number, label2: string) -> () ) -> ()``
you want ``(targetFoo: Foo: Player, label1: number, label2: string) -> ()``.
