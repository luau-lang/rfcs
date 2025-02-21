# `keyof` and `rawkeyof` type functions

**Status**: Implemented

## Summary

This RFC proposes the addition of two type functions, `keyof` and `rawkeyof`,
which can be used to derive a type automatically for the keys of a table or
class.

## Motivation

The primary motivation of this proposal is to make it easier to work with tables
as objects and/or enumerations, and to reduce the amount of duplicate work users
must undergo in order to type code that does this. For instance, consider the
following example code:

```luau
type AnimalType = "cat" | "dog" | "monkey" | "fox"

local animals = {
    cat = { speak = function() print "meow" end },
    dog = { speak = function() print "woof woof" end },
    monkey = { speak = function() print "oo oo" end },
    fox = { speak = function() print "gekk gekk" end }
}

function speakByType(animal: AnimalType)
    animals[animal].speak()
end

speakByType("dog") -- ok
speakByType("cactus") -- errors
```

This code is totally reasonable, but we had to manually write a type that lists
out the keys of the table `animals` in order to get type safety. The larger the
table is the less tractable this becomes, and it'd be easy to imagine a user
opting to instead provide the annotation `string` and just losing type safety in
this situation.

## Design

The solution to this problem is a type function, `keyof`, that can compute the
type based on the type of `animals`. This would allow us to instead write this
code as follows:

```luau
local animals = {
    cat = { speak = function() print "meow" end },
    dog = { speak = function() print "woof woof" end },
    monkey = { speak = function() print "oo oo" end },
    fox = { speak = function() print "gekk gekk" end }
}

type AnimalType = keyof<typeof(animals)>

function speakByType(animal: AnimalType)
    animals[animal].speak()
end

speakByType("dog") -- ok
speakByType("cactus") -- errors
```

Now, regardless of how the `animals` table grows, `AnimalType` will always be
defined as the type of indices into the table. At its base, this is a simple
solution to a simple, but common problem of code duplication for types.

Unfortunately, there are some edgecases to be concerned about because of
metatables and the `__index` metamethod in particular. There's valid arguments
for both including properties available from `__index` as well as for excluding
them. This RFC proposes that there are two reasonable solutions, which
correspond directly to the runtime operations `t[i]` and `rawget(t, i)`. The
former is an instance where you'd want the type to incorporate `__index`
properties appropriately, and the latter is a case where you would not. So, we
analogously provide both `keyof<T>` and `rawkeyof<T>` which provide all the
legal keys for indexing and only the keys legal for `rawget` respectively.

So, if we consider some very simple strawman code here:

```luau
local MyClass = { Foo = "Bar" }
local OtherClass = setmetatable({ Hello = "World" }, { __index = MyClass })

type MyClass = typeof(MyClass)
type OtherClass = typeof(OtherClass)
```

`keyof<OtherClass>` will give you `"Foo" | "Hello"` while `rawkeyof<OtherClass>`
will give you only `"Hello"`. This would then let you use indexing and `rawget`
appropriately in a type-safe way for the motivating style of dynamic code.

The remaining bit of complexity is the question of what to do for types that do
not necessarily have one consistent set of keys. For instance, if you consider
the type `{ x: number, y: number } | { a: number, y: number }`, you might wonder
what you would get out of `keyof`. One reasonable answer is `"y"` since that is
the greatest common subset of keys present in the components of the union.
Another reasonable, albeit more conservative, answer is to simply say that the
operator fails to resolve in this situation. This RFC proposes that we return
the greatest common subset of keys since this corresponds to the set of keys
that are allowed by indexing operations on tables of that type.

## Drawbacks

The main drawbacks of implementing this are that it requires some pretty
powerful machinery to support properly. Fortunately, however, we've already
built the general machinery to support type functions into the ongoing work on
the new type inference engine for Luau, and as such, there is little remaining
drawback to implementing this. In fact, the implementation is already all done
in the new type inference engine and amounts to less than 200 lines of code
including comments. So, beyond looking for motivating examples for potentially
computing the greatest common subset of keys for unions of tables and the small
amount of work that implementing that might entail, the author of this RFC hopes
that this effort is simply an easy win for OOP in Luau supported by the
technical credit built up by implementing the new type inference engine.

## Alternatives

The main alternative designs are, in a sense, discussed in the design section.
We could simply choose to offer only one of `keyof<T>` and `rawkeyof<T>` in
order to keep things simpler, but both have corresponding operations in the
language, and seem useful, and almost all of the work to implement them is
shared. So, doing less here doesn't really save us anything. The other
alternative is surrounding the choice of what to do when the set of keys are not
identical across unions of tables. The proposal here was to provide the greatest
common subset of keys, but the alternative of failing to reduce and thus
producing a type error seems perfectly reasonable as well.
