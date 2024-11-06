# Metatable Type Functions

## Summary

Implement type functions for `getmetatable` and `setmetatable`.

## Motivation

### `setmetatable` Type Function

There is currently no way for users to apply metatable type information to a type without the usage of `typeof()`. This isn't ideal, as it adds verbosity and boilerplate to common patterns such as object-oriented programming. For example, the following:

```lua
local clock = {}
type Identity = typeof(setmetatable({} :: { time: number }, { __index = clock }))
```

could be reduced to:

```lua
local clock = {}
type Identity = setmetatable<{ time: number }, { __index: clock }>
```

### `getmetatable` Type Function

[Issue #1435](https://github.com/luau-lang/luau/issues/1435) is caused by a lack of special behavior for `getmetatable` when the type's metatable has a `__metatable` field. This is fixable in a variety of ways, however a type function has multiple benefits and exposes more type-level expression for users. It fits in nicely with the new solver, as type functions are a cleaner alternative to magic functions.

## Design

### `setmetatable` Type Function

In the following code example, `Identity` should evaluate to `{ sound: string, @metatable: { __index: animal } }`:

```lua
local animal = {}
type Identity = setmetatable<{
	sound: string
}, {
	__index: animal
}>
```

### `getmetatable` Type Function

In the following code example, `ReversedIdentity` should evaluate to `{ __index: animal }`:

```lua
local animal = {}
type Identity = setmetatable<{
	sound: string
}, {
	__index: animal
}>

type ReversedIdentity = getmetatable<Identity>
```

Due to `__metamethod`, additional behavior needs to be met. In the following code example, `MetatableType` should evaluate to `"No metatable here!"`:

```lua
type metatable = {
	__metatable: "No metatable here!"
}

type MetatableType = getmetatable<setmetatable<{}, metatable>>
```

## Drawbacks

Introducing a large library of "global" type functions can clutter naming space for users.

The name of `setmetatable` has an interpretation of implying a side effect; tables are references, and `setmetatable` is a side-effecting function. It could be argued that `setmetatable` is not a good name for a type function which has no side effects. However, consistency is desirable. A slight rename of a global function could cause confusion, and likely cause mistakes to be made. Type functions also have a precedent of not allowing side effects, so therefore a `setmetatable` type function would not have ambiguity when it comes to side effects.

## Alternatives

Do nothing, and rely on unique metatable type syntax to achieve what a `setmetatable` type function would achieve. However, this still has the issue of `getmetatable`. It is flat out incorrect at the moment, and a type function would be an ideal solution to that problem. If a type function is introduced, then any user could reasonably expect a counterpart `setmetatable` type function. Both metatable type functions and metatable syntax can exist at the same time, and that might be ideal.

Do nothing. `typeof` solves the current issues with accessibility to the current object-oriented programming patterns, and the issue relating to getmetatable can be solved by overloads.