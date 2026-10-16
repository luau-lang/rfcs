# Names in Type Packs

## Summary

Allowing users to always provide names within type packs ``(T...)``, so that parameter names can be attached to types.
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
There is no context or indication on what these provided types ``(number, string)`` are meant to represent.
You don't want to use a function annotation to fill in ``T...`` because we want ``(self: WrappedSignal<T...>, targetFoo: Foo, number, string)``
and not ``(self: WrappedSignal<T...>, targetFoo: Foo, (number, string) -> ())``.

But you can see that ``targetFoo: Foo`` is present, but only because function annotations specifically let you describe argument names, which APIs can pick-up on.

<br/>

Luau already allows you to provide parameter names within function annotations.
```luau
(x: number) -> ()
```

But when generic type packs are used, this information is no longer available.
```luau
type Callback<T...> = (T...) -> ()
```

``T...`` will only emit over types but not any names/labels.

And currently, there isn't a way to provide a name/label when using ``Callback<T...>``


## Design


The idea is to allow to describe names in any valid type pack context.

Allowing you to do ``Type<(x: number)>`` if the base is ``Type<T...>`` without being exclusively restricted to function annotations anymore.

Names/Labels are only metadata:
- Names/Labels are NOT enforced in any type checking way
- They don't affect the type itself
- They only improve front-end hints
- Nothing changes about the types own identity, it just gets associated with a name.
- The name is NOT forced and are completely optional.
- Names propagate through type aliases.

### Usage and Examples:

```luau
type Callback<T...> = (T...) -> ()

function Foo(cb: Callback<(targetBar: Bar)> end
-- cb shows (targetBar: Bar) -> ()
```

```luau
type Bar<T> = (T) -> ()
type A = Bar<(num: number)> -- not valid, '(T)' is not a type pack!
type MyNumber = (num: number) -- not valid not a type pack!
```


```luau
type Foo<T...> = (T...) -> ()

-- Names are optional
type A = Foo<(a: number, number, label: number)> -- allowed

type B = Foo<(x: number), (y: number)> -- not valid, they're not type packs
type C = Foo<(x: number, y: number)> -- valid

type D = (num: number) -> () -- already works in Luau, no changes!
type E = () -> (num: number) -- valid
```

```luau
type Mixed<A, T...> = (A, T...) -> ()
type B = Mixed<string, (x: number, y: number)> -- valid, T... is a type pack, A is not
type B = Mixed<(text: string), (x: number, y: number)> -- not valid "A", is not a type pack
```

<br/>

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



<br/>


```luau
type Func = () -> (x: number, y: number, z: number)
function Func2(): (a: number, b: number, c: number)
  return 1,2,3
end
```





## Drawbacks

- Most likely none, other than the implementation.
- Union or intersections may keep a random name.


This RFC doesn't solve that we can't rename ``T`` here.
```luau
type Foo<T> = (arg: T) -> ()
type A = Foo<(x: number)> -- (x: number), does not count as a type pack

-- Not possible!
type Bar<T,U> = (T,U) -> ()
local e: Bar<(x: number, y: number)>
```


## Alternatives

- A new syntax where it would be ``[x: number, y: number, z: number]`` or ``[x: number], [y: number], [z: number]``, that can be applied into types directly, other than within the type pack.
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
