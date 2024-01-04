# Polymorphic Table Types

## Summary

We propose implementing polymorphic tables of the form `<T>{ [F<T>] : G<T> }` analogous to already-existing polymorphic functions `<T>(F<T>) -> G<T>`.

## Motivation

The goal of this RFC is furthering type-safety and type-representation, especially in niche or performance-critical applications. polymorphic functions exist already in the form `<T>(F<T>) -> G<T>`, where `T` is inferred at the call site, with similar applications.

At a type level, luau functions and tables are homomorphic as type mappings. One can view a table as a function mapping its keys to its values, or a function as a table with inputs as keys and outputs as values. This change resolves the hole left behind implementing polymorphic functions while excluding polymorphic tables, forcing type-safe applications to prefer functions over tables as maps.

### General Accessor Pattern

At an application level, the current workaround for this is to wrap the table in a function getter like so:

```lua
-- private
local foo = {} :: { [F<any>]: G<any> }

-- public
local function getFoo<T>(k: F<T>): G<T>
	return foo[k]
end

local function setFoo<T>(k: F<T>, v: G<T>)
	foo[k] = v
end

local iterFoo = pairs(foo)
```

This has five immediate consequences.

1. This pattern extends to every table of this nature, increasing surface area, duplication, and maintenance.
2. In the event the functions don't get inlined, we incur an extra performance penalty per read and write, disallowing certain applications type-safety.

If we are using a function then one might try and take advantage of the extra indirection by hiding the table implementation to prevent misuse. This raises the following consequences:

3. The implementation is opaque and can be modified later without worry.
4. Mutations require yet another function, a setter, with the cons listed before.
5. Iterations requires yet another function, or at least stored iterator, reinforcing the cons listed before.

In short, tables are mutable and iterable, functions are not.

At this point if one is not satisfied with the extra baggage that goes behind a type-safe access or write of this nature, they will simply lie to the type checker or compromise and move on. This is exactly what we wish to avoid.

This proposal targets users already neck-deep in the type-system supporting libraries and tools, unable to achieve safety without compromising performance or their api and affecting thousands of users, or the occasional type-savvy user getting that perfect type for their data structure and feeling accomplished.

### Specific Accessor Pattern

We'll demonstrate using a minified ECS api from a library currently in production, getting components from an entity. This library depends on accessing components from tables using their constructors as indexers. The constructors contain the type information of the component, so it is theoretically possible to infer the type of the resulting component from the constructor, but this is not possible as of now because polymorphic tables don't exist.

```lua
-- <C>(() -> C) -> Factory<C>
world.factory

-- <'Foo'>(() -> 'Foo') -> Factory<'Foo'> ==> Factory<'Foo'>
local A = world.factory(function()
	return 'Foo'
end)

-- <'Bar'>(() -> 'Bar') -> Factory<'Bar'> ==> Factory<'Bar'>
local B = world.factory(function()
	return 'Bar'
end)

-- number
local e = world.entity()

-- 'Foo'
A.add(e)

-- 'Bar'
B.add(e)

-- { [Factory<any>]: any? } 	(no possible way to extract the generic)
local components = world.get(e)

-- { [Factory<any>]: any? }[Factory<'Foo'>] ==> any?
local a = components[A]

-- { [Factory<any>]: any? }[Factory<'Bar'>] ==> any?
local b = components[B]
```
We've lost our type information of `a` and `b` in the last table step!
- "Why not use a getter?"
  - Mentioned before, API bloat or performance concerns, getting components from this table usually happens during iterations and thousands of times a frame.

- "Why not implement better metamethod inference?"
  - The `components` table guarantees users may define their own metatables. Metatables are not an elegant, reliable, or wholly-acceptable solution to this lack of types. However better inference is always a good thing! This is a great idea for a *different* RFC.

## Design

We propose naturally extending this type-mapping capability to tables in the backwards-compatible form `<T>{ [F<T>]: G<T> }`. We recycle our familiar syntax of polymorphic functions and use them in a very similar fashion. If one understands polymorphic functions, they will understand this as well. It is backwards compatible as it is currently invalid syntax.

To rewrite our previous code examples using the new type:

```lua
local foo = {} :: <T>{ [F<T>]: G<T> }
```

This is not only simpler code, but shorter without all the duplication or extra api that comes with such a structure. Looking at the syntax, it reads very similarly to if it were a function:

```lua
type A = <T>{ [F<T>]: G<T> }
type B = <T>(F<T>) -> G<T>
```

Back to the ECS example, we change the type of `components` to be what it should have been all along:

```lua
-- <C>{ [Factory<C>]: C? }		(look we extracted the generic!)
local components = world.get(e)

-- <'Foo'>{ [Factory<'Foo'>]: 'Foo'? }[Factory<'Foo'>] ==> 'Foo'?
local a = components[A]

-- <'Bar'>{ [Factory<'Bar'>]: 'Bar'? }[Factory<'Bar'>] ==> 'Bar'?
local b = components[B]
```

And perfect intellisense is now achieved without a single typecast or getter (per component). It was simple too, encouraging the simplest solution in the process.

We believe that this implementation is a natural step in the direction towards a fuller Luau type-system, as it already has a functional cousin and practical use-cases. Concerns with api coherency, stylistic coherency, learning curve, acceptance, and more are all already answered with polymorphic functions. If you want to understand the behavior of a polymorphic table, simply look to the behavior of polymorphic functions as they will have the exact same properties.

`function` $\cong$ `table`

```lua
<T>(any) -> T implies T = never
<T>{ [any]: T } implies T = never
```
```lua
<T>(any) -> (T) -> G<T> implies T is inferable
<T>{ [any]: { [T]: G<T> } } implies T is inferable
```

## Drawbacks

This change may include hidden complexities at an implementation level, but we imagine some relief as aspects of this feature have been implemented before and may potentially be recycled. In this same vein, there is not a worry of hurting the type solver's performance with this feature.

There is little to no concern with feature-creep, as it is a necessity to achieve certain patterns used today while retaining type-safety, for the same reasons polymorphic functions exist in their current form.

This feature does complicate the type solver and language, however it is done in the best case scenario as an opt-in complication only by those that need it, typically by tool maintainers and almost never typical users who are not well versed with luau types.

Native code generation will most likely not be able to optimize any more than it would `{ [any]: any }`.

## Alternatives

The alternatives presented, lying and compromising or using polymorphic functions, have real and lasting effects that negatively contribute towards codebases in this predicament.

Mentioned previously, increasing `__index`'s inference capabilities does allow type inference, but it also comes at the cost of changing how source code is written and a metatable which is not always an option. This is still a good idea however, and should be brought up in another RFC!
