# Syntax for table property access modifiers

**Status**: Implemented

## Summary

We need syntax to match the semantics of read-only and write-only modifiers for table properties and indexers.

## Motivation

See the semantic RFCs for motivation:

* [Read-only table properties](https://github.com/luau-lang/rfcs/blob/master/docs/property-readonly.md)
* [Write-only table properties](https://github.com/luau-lang/rfcs/blob/master/docs/property-writeonly.md)

## Design

We will use the following syntax for describing a read or a write type of a property:

```luau
type ReadOnly = { read x: number }
type WriteOnly = { write x: number }
```

A property will occasionally be both readable and writable, but using different
types.  The author will have to duplicate the property name in this case:

```luau
type Foo = {
    read p: Animal,
    write p: Dog
}
```

The tokens `read` and `write` are contextual.  They are still valid property names.

```luau
type Reader = { read: () -> number }
type Writer = { write: (number) -> () }
```

Indexers can also be read-only or write-only.

```luau
type ReadOnlyMap<K, V> = { read [K]: V }
type WriteOnlyMap<K, V> = { write [K]: V }

type ReadAnimals = { read Animal }
type WriteDogs = { write Dog }
```

Mixed indexers are allowed but heavily discouraged:

```luau
type MixedMap = { read [string]: Animal, write [string]: Dog }
type MixedArray = { read Animal, write Dog }
```

Redundant record fields are still disallowed: Each field may have at most one
read type and one write type:

```luau
type A = { read x: string, write x: "Hello" } -- OK
type C = { read x: string, read x: "hello" }  -- ERROR
type B = { x: string, read x: "hello" }       -- ERROR
```

We place no restriction on the relationship between the read and write type.
The following is certainly a bad idea, but it is legal:

```luau
type T = { read n: number, write n: string }
```

This syntax is readable, matches the flavour of preexisting Luau syntax well,
and is pretty easy to parse efficiently.

## Drawbacks

We expect it to be very commonplace to have table types containing read-only
functions.  This syntax is a little bit verbose and maybe unintuitive for that
use case.

`read` and `write` are also very useful method names.  It's a little bit
awkward to talk about a table that has a `read` or a `write` method:

```luau
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

```luau
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

```luau
  type Vector2 = { @read x: number, @read y : Number }
```

For the other proposals, there are four possibilities, depending on whether the
modifier is west-coast or east-coast, and whether it modifies the propertry name
or the type:

```luau
  type Vector2 = { read x : number, read y : number }
  type Vector2 = { x read : number, y read : number }
  type Vector2 = { x : read number, y : read number }
  type Vector2 = { x : number read, y : number read }
```

The east-coast options are not easy-to-read with names, but are
easier with symbols, especially since `T?` is already postfix, for
example

```luau
  type Foo = { p: number?+ }
```

### How do we talk about properties with differing read- and write-types?

One corner case is that type inference may deduce different read- and
write-types, which need to be presented to the user. For example the
read-type of `x` is `Animal` but its write-type is `Dog` in the principal type of:

```luau
   function f(x)
     let a: Animal = x.pet
     x.pet = Dog.new()
     return a
   end
```

If we are adding the modifier to the property name, we can repeat the name, for example

```luau
  x : { read pet : Animal, write pet : Dog }
```

If we are adding the modifier to the property type, we can give both types, for example:
```luau
  x : { pet : read Animal + write Dog }
```

This syntax plays well with symbols for modifiers, for example
```luau
  x : { pet : +Animal -Dog }
```
