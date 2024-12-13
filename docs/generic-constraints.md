# Generic Constraints
## Summary
Introduce syntax to extend/constrain the type of a generic.
## Motivation
Luau currently does not provide a way in order to constrain the type of a generic without direct assertions.
```lua
local qux = { foo = 10, bar = "string" }

local function getProperty<T>( object: T, key: keyof<T> )
  return object[key]
end

local foo = getProperty( qux, "foo" )
-- foo: number | string
local bar = getProperty( qux, "bar" )
-- bar: number | string
```
This is wrong! We would expect foo to be a number, with bar being a string.
In the following snippet, it is impossible to tell whether or not the key variable is the same as the one being used to index T in the callback.
```lua
local function callbackProperty<T>( object: T, key: keyof<T>, callback: (index<T, keyof<T>>) -> () )
...
```
## Design
The design of this would take inspiration from TypeScript's `extends`, where instead of the additional keyword, we would use `&`.
```lua
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
The following snippet would also thus be correctly inferred.
```lua
local qux = { foo = 10, bar = "string" }

local function callbackProperty<T, K & keyof(T)>( object: T, key: K, callback: (index<T, K>) -> () )
  callback( object[key] )
end

callbackProperty( object, "foo", function( foo )
  -- foo: number -- this is expected!
end)
```
An alternative syntax could be the following, but does not satisfy current requirements about `:` only being used inside `()` and `{}`, and personally does not look as good.
```lua
local function getProperty<T, K: keyof<T>>( object: T, key: K ) ... end
local function callbackProperty<T, K: keyof(T)>( object: T, key: K, callback: (index<T, K>) -> () ) ... end
```
## Drawbacks
- I am not personally familiar with the internals of the typechecker, but this has a chance to further complicate type inference.
- Adding an extra use to `&` could make its usage more confusing to novices.
## Alternatives
- Don't do this; this would make it impossible for functions like above to be able to be inferred correctly. Just let people explicitly type their variables instead of inferring types. This makes code more verbose and would likely not allow for full optimization.
- Use overloaded functions as previously mentioned, but this wouldn't allow the usage of a generic, would require users to add a new overload for each key.
