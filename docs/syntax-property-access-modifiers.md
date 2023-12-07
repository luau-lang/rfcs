# Syntax for table property access modifiers

## Summary

We need syntax to match the semantics of read-only and writed-only modifiers for table properties and indexers.

## Motivation

See the semantic RFCs for motivation:

* [Read-only table properties](https://github.com/luau-lang/rfcs/blob/master/docs/property-readonly.md)
* [Write-only table properties](https://github.com/luau-lang/rfcs/blob/master/docs/property-writeonly.md)

## Design

We will use the following syntax for describing a read or a write type of a property:

```lua
type ReadOnly = { read x: number }
type WriteOnly = { write x: number }
```

A property will occasionally be both readable and writable, but using different
types.  The author will have to duplicate the property name in this case:

```lua
type Foo = {
    read p: Animal,
    write p: Dog
}
```

The tokens `read` and `write` are contextual.  They are still valid property names.

```lua
type Reader = { read: () -> number }
type Writer = { write: (number) -> () }
```

Indexers can also be read-only or write-only.

```lua
type ReadOnlyMap<K, V> = { read [K]: V }
type WriteOnlyMap<K, V> = { write [K]: V }

type ReadAnimals = { read Animal }
type WriteDogs = { write Dog }
```

Mixed indexers are allowed but heavily discouraged:

```lua
type MixedMap = { read [string]: Animal, write [string]: Dog }
type MixedArray = { read Animal, write Dog }
```

This syntax is readable, matches the flavour of preexisting Luau syntax well,
and is pretty easy to parse efficiently.

## Drawbacks

We expect it to be very commonplace to have table types containing read-only
functions.  This syntax is a little bit verbose and maybe not intuitive for that
use case.

`read` and `write` are also really useful method names.  It's a little bit
awkward to talk about a table that has a `read` or a `write` method:

```lua
type Reader = { read read: () -> number }
type Writer = { read write: (number) -> () }
```

It is important to consider that this will be an issue for any keywords we might
choose unless we were to take the step of picking something unlikely to be a
useful property name. (ie something weird looking and ugly)

Lastly, mixed indexers are very awkward both in syntax and semantics.

## Alternatives

The design space for syntax includes:

* Names, symbols or attributes?
* Modifier position?
* How to parse or serialize properties with differing read- and write-types?

### Names, symbols or attributes?

We could use names for modifiers, such as

* `get` and `set`
* `const` and `mut`
* `read` and `write`
* `readonly` and `writeonly`

One issue is that these are all valid identifiers, so if we want
backward compatbility, they cannot be made keywords. This presents
issues with code that uses the chosen names as type or property names,
for instance:

```lua
  type set = { [any] : bool }
  type ugh = { get set : set }
```

We could use symbols, for example

* `+` and `-`
* (Are there other obvious pairs of symbols?)

We could use attributes, for example

* `@read` and `@write`
* ... as per "names for modifiers", only prefixed by `@` ...

These both have the advantage of being unambiguous and easier to parse. Symbols
are terser, whch is both good and bad.

We decided not to use glyphs because they are more difficult to understand,
don't contribute very much, and aren't very stylistically consistent with other
Luau syntax.

### Modifier position?

For attributes, the position is given by the syntax of attributes, for example:

```lua
  type Vector2 = { @read x: number, @read y : Number }
```

For the other proposals, there are four possibilities, depending on whether the
modifier is west-coast or east-coast, and whether it modifies the propertry name
or the type:

```lua
  type Vector2 = { read x : number, read y : number }
  type Vector2 = { x read : number, y read : number }
  type Vector2 = { x : read number, y : read number }
  type Vector2 = { x : number read, y : number read }
```

The east-coast options are not easy-to-reade with names, but are
easier with symbols, especially since `T?` is already postfix, for
example

```lua
  type Foo = { p: number?+ }
```

### How to parse or serialize properties with differing read- and write-types?

One corner case is that type inference may deduce different read- and
write-types, which need to be presented to the user. For example the
read-type of `x` is `Animal` but its write-type is `Dog` in the principal type of:

```lua
   function f(x)
     let a: Animal = x.pet
     x.pet = Dog.new()
     return a
   end
```

If we are adding the modifier to the property name, we can repeat the name, for example

```lua
  x : { read pet : Animal, write pet : Dog }
```

If we are adding the modifier to the property type, we can give both types, for example:
```lua
  x : { pet : read Animal + write Dog }
```

This syntax plays well with symbols for modifiers, for example
```lua
  x : { pet : +Animal -Dog }
```
