# Names in Type Packs

## Summary

Allowing users to always provide names within type packs, such as ``(x: number)`` syntax, would allow users to attach parameter names to a within a type pack.
This improves IntelliSense and autocomplete, and allows custom callbacks to show you argument names instead of just the type.

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

But in the IntelliSense case, a developer would immediately know what each of these type _represents_. In this case ``x``, ``y`` and ``z``.

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

``T...`` will only emit over types but not any names/labels.

And currently, there isn't even syntax to provide a label when using ``Callback<T...>``



Why are we doing this? What use cases does it support? What is the expected outcome?

## Design

This syntax here ``(number)`` is a type pack.

And the idea is to allow to provide names in any type pack syntax.

e.g. allowing you to do ``(x: number)`` without being exclusively restricted to function annotations anymore.

Names/Labels are only metadata:
- Names/Labels are NOT enforced in any type checking way
- They don't affect the type itself
- They only improve front-end hints


### Example:

```luau
type Callback<T...> = (T...) -> ()

function Foo(cb: Callback<(targetBar: Bar)> end
-- cb shows (targetBar: Bar) -> ()
```

```luau
type Foo<T...> = (T...) -> ()
type Bar<T> = (T) -> ()

type A = Foo<(a: number, number, label: number)> -- allowed
type B = Foo<(x: number), (y: number)> -- allowed

type C = Bar<(x: number, y: number)> -- Error: More than one type!
type D = (num: number) -> () -- Already works in Luau, no changes!
```

<br/>

### Usage:
```luau
type WrappedSignal<T...> = {
	FireToFoo: (self: WrappedSignal<T...>, targetFoo: Foo, T...) -> (),
}

type MySignal = WrappedSignal<(amount: number, message: string)>

--[[
  MySignal.FireToFoo would become
    (self: WrappedSignal<T...>, targetFoo: Foo, amount: number, message: string) -> ()
]]
```

Nothing changes about the types own identity, it just gets associated with metadata.

<br/>


```luau
type MyNumber = (x: number)
type Func = () -> (x: number, y: number, z: number)
```

In the event where an annotation already has a name. The name would get overwritten if...
```luau
type Foo<T...> = (args: T...) -> ()
type Foo_2<T> = (arg: T) -> ()

type A = Foo<(number, number)> -- nothing changes about "args"
type A = Foo<(x: number, y: number)> -- "args" would be gone if something substituted a generic
type C = Foo_2<(x: number)> -- same here for "arg"
```



## Drawbacks

- Most likely none, other than the implementation.
- Whether ``(arg: T)`` should have ``"arg"`` replaced or not, if substituted by ``(x: number)``.


## Alternatives

- A new syntax where it would be ``[x: number, y: number, z: number]`` or ``[x: number], [y: number], [z: number]``
<br/>

- Type functions could expose modifying argument names, but that would only work for function annotations, not on a type alone.
There wouldn't be a way to attach a label to a type.

```luau
type WrappedSignal<T...> = { FireToFoo: (self: WrappedSignal<T...>, targetFoo: Foo, T...) -> (); }
type MySignal = WrappedSignal<(label1: number, label2: string) -> ()>
```
And this would also mean that ``WrappedSignal`` needs to implement a type function now, that takes out the argument names and transforms ``.FireToFoo``.
And that sounds too complicated.


You don't want it to be ``( targetFoo: Foo: Player, (label1: number, label2: string) -> () ) -> ()``

You want ``(targetFoo: Foo: Player, label1: number, label2: string) -> ()``.
