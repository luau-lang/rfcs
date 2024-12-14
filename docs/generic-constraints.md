# Generic Constraints
## Summary
Introduce syntax to constrain the type of a generic.
## Motivation
Luau currently does not provide a way in order to constrain the type of a generic.
```luau
local qux = { foo = 10, bar = "string" }

local function getProperty<T>( object: T, key: keyof<T> ): index<T, keyof<T>>
	return object[key]
end

local foo = getProperty( qux, "foo" )
-- foo: number | string
local bar = getProperty( qux, "bar" )
-- bar: number | string
```
Type interference believes that either value could be a number or string as `keyof<T>` is a union type of either foo or bar.
```luau
local function callbackProperty<T>( object: T, key: keyof<T>, callback: (property: index<T, keyof<T>>) -> () )
....
```
It is impossible to tell whether or not `key` is the same variable being used to index `T` in the callback, and thus Luau infers the type of `property` to be `number | string`.
```luau
local function callbackProperty<T>( object: T, key: keyof<T>, callback: (property: index<T, K>) -> () )
....
```
This does not work in Luau.
> Ideally, this behaviour should work out of the box, but, could be solved with bounded polymorphism.

## Design
### Syntax
There are a few options for the syntax of generic constraints.
1. Treat the generic parameter as a variable--annotate it as declaring a new variable.
```luau
local function getProperty<T, K: keyof<T>>( object: T, key: K ): index<T, K>
	return object[ key ]
end

type getProperty<T, K: keyof<T>> = ( object: T, key: K ) -> index<T, K>
```
This would match the current expectation of `:` being used to type annotate a variable. This would be backwards compatible without major performance implications. As a note, the ordering of `T` and `K` is arbitrary and can be switched if desired.

2. Use a `where` clause.
```luau
local function getProperty<T, K where K: keyof<T>>( object: T, key: K ): index<T, K>
	return object[ key ]
end

type getProperty<T, K where K: keyof<T>> = ( object: T, key: K ) -> ( index<T, K> )
```
This would allow users to specify the constraints of the separately from the generic declaration itself. This would be reminiscent to users of C#. Imposing multiple constraints on different generics can be done by delimiting with `,`. This would be backwards compatible, without major performance implications, as it could just be a conditional keyword. This could be used in conjuction with option 1.
> This isn't nearly as elegant for smaller types compared to option 1, but would be incredibly powerful for generics with lots of constraints. This allows for neatly distributing the declaration along multiple lines.

### Usage
#### Subtyping Constraints
This is the problem posed at the beginning of this RFC.
```lua
local function callbackProperty<T, K: keyof<T>>( object: T, key: K, callback: (property: index<T, K>) -> () )
	callback( object[ key ] )
end
```
`K` will be constrained to be `keyof<T>`. This would allow `index<T, K>` to infer the type properly.
#### Equality Constraints
```luau
-- Attempt 1
local function sumSpecific( a: number | vector, b: number | vector ): number | vector
	return a + b -- not okay!
end
-- Attempt 2
local function sumSpecific<T>( a: T, b: T ): T
	return a + b -- not okay! not all types are addable.
end
```
This is to be expected. Luau does not interpret `a` and `b` as the same type, but if we use a generic then Luau throws an error that not all types are addable.
```luau
-- Attempt 3
local sumSpecific: ( ( number, number ) -> ( number ) ) & ( ( vector, vector ) -> ( vector ) ) = function( a, b )
	return a + b -- okay!
end
```
This is fine, but is not concise whatsoever, and does not allow for the traditional function declaration syntax.
```luau
local function sumSpecific<T: number | vector>( a: T, b: T ): T
	return a + b
end

sumSpecific( vector.create( 42, 42, 42 ) + vector.create( 143, 143, 143 ) ) -- okay!
sumSpecific( 42, 143 ) -- okay!

sumSpecific( vector.create( 42, 42, 42 ), 143 ) -- error! a and b do not match.
```
In this example, `T` must either be a number or vector, and setting annotating variables `a` and `b` as `T` would solve this. This can also be done by overloading the function but is more verbose and less elegant than this solution. This cannot be done with Luau's `add` type function, as it would allow all 'addable' types, and could not guarantee the return type being the same as both inputs.

## Drawbacks
- There are no inequality constraints nor not-subtyping constraints.
	> This drawback is difficult justify in Luau's current state. The introduction of type negation could solve this, but is outside the scope of this RFC. If type negation were to be added, the proposed syntax would allow for both of these types of constraints.
- This would complicate Luau's grammar and add an extra keyword.
	> Any addition would add further complexity to the language. Adding the `where` keyword would also introduce another conditional keyword to the language.
- If user-defined type functions supported generics, it would have to be expanded to support type constraints.
	> This has major backwards compatibility implications depending on how user-defined type functions handle generics in the future. This won't be a problem if bounded polymorphism is implemented before that.

## Alternatives
1. Don't do this; this would make it impossible for functions like above to be able to be automatically inferred correctly. Just let people explicitly annotate their variables instead of inferring types. This makes code more verbose.
	> This would disallow for code that specifically makes use of generics to automatically output a response.
2. Manually write verbose overloaded functions types.
	> This suffers the same pitfalls as alternative 1, and does not allow for code to be easily expandable. An example can be found as Attempt 3 of Equality Constraints.
