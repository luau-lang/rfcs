# Generic Constraints
## Summary
Introduce syntax to extend/constrain the type of a generic.
## Motivation
Luau currently does not provide a way in order to constrain the type of a generic.
```luau
local qux = { foo = 10, bar = "string" }

local function getProperty<T>( object: T, key: keyof<T> )
  return object[key]
end

local foo = getProperty( qux, "foo" )
-- foo: number | string
local bar = getProperty( qux, "bar" )
-- bar: number | string
```
Type interference believes that either value could be a number or string as keyof<T> is too broad.

```luau
local function callbackProperty<T>( object: T, key: keyof<T>, callback: (index<T, keyof<T>>) -> () )
....
```
It is impossible to tell whether or not the key variable is the same as the one being used to index T in the callback, and thus Luau infers that it could be a number | string.

## Design
The design of this would take inspiration from TypeScript's `extends`, where instead of the additional keyword, we would use `&`.
```luau
local qux = { foo = 10, bar = "string" }

local function getProperty<T, K & keyof<T>>( object: T, key: K )
  return object[key]
end

local foo = getProperty( qux, "foo" )
-- foo: number
local bar = getProperty( qux, "bar" )
-- bar: string
```
This would correctly infer the type of the each key's value.
```luau
local qux = { foo = 10, bar = "string" }

local function callbackProperty<T, K & keyof<T>>( object: T, key: K, callback: (index<T, K>) -> () )
  callback( object[key] )
end

callbackProperty( qux, "foo", function( foo )
  -- foo: number -- this is expected!
end)
```
This would also be correctly inferred as index<T, K> would correctly result in the type of the field.

An alternative syntax could use `:` instead of `&`. However, this would not match the current semantics of `:` and using `&` implies a union of types.
```luau
local function getProperty<T, K: keyof<T>>( object: T, key: K ) .... end
local function callbackProperty<T, K: keyof<T>>( object: T, key: K, callback: (index<T, K>) -> () ) .... end
```
## Drawbacks
- I am not personally familiar with the internals of the typechecker, but this has a chance to further complicate type inference.
- Adding an extra use to `&` could make its usage more confusing to novices.
## Alternatives
- Don't do this; this would make it impossible for functions like above to be able to be inferred correctly. Just let people explicitly type their variables instead of inferring types. This makes code more verbose and would likely not allow for full optimization.
- Use overloaded functions as previously mentioned, but this would not allow the usage of a generic, and would thus require users to add a new overload for each key.
