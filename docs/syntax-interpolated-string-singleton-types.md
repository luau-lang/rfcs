# Interpolated String Singleton Types

## Summary

Introduce string interpolation syntax for string singleton types (stringletons hereon), and bindings which currently
generalize to stringletons will do so for interpolation string expressions if all sub-expressions are singleton types.

## Motivation

Luau has four types of string expressions:

```luau
-- single quoted
const nyanya = 'mew'
-- double quoted
const mrrp = "chirp"
-- long
const meow = [[
purr
hiss
]]
-- interpolated
const kya = `claw {mrrp} {nyanya}`
```

Currently, stringleton type annotations are supported in the form of single-quoted, double-quoted, and
long strings - but not interp-strings. This makes type syntax inconsistent with runtime expressions, and makes it very
difficult to combine string types. An existing solution exists by creating a user-defined type function to concatenate
string singletons. However, access to basic formatting in the type system would provide:


- syntactic consistency; the benefit here is obvious. Users of the language may be surprised or confused that they
cannot use interpolation strings inside type annotations.

- ease-of-use; Some style guides or formatters may prefer interpolated strings by default for cosmetic purposes or
because it is faster to add a sub-expression than to change the quote style of a string, deal with escaping, etc..., and
the same applies to type syntax as well.

- higher quality inference; This is an implementation detail, but user-defined type functions also currently block
bi-directional type inference. Although they may improve in the future, it is fundamentally impossible to reason about
the result of a user-defined type function as empirically as you can with built-in types. Although admittedly uncertain
& abstract, it's possible future improvements to polymorphic type signatures will add usefulness to interpolated string
types.

## Design

### Syntax

Interpolation quotes will become valid, usable stringleton syntax:

```luau
type Yowl = `growl :3`
```

It will also support sub-expression types:

```luau
type Headbutt = "Hello"
-- 'Hello World'
type PupilsDilated = `{Headbutt} World`
```

### General Purpose Stringification & Inference

Exposing any kind of literal stringification to the type system would be fragile in concept because any interface we use
would need to be completely stable. We probably don't want `__tostring` because runtimes can (and do) modify that. We
probably do not want to expose the type system's `ToString.h` because it is disjoint from the runtime type. The `print`
global is even less standardized. Returning 'string' would be arduous to debug - so non-singleton sub-expressions will
instead report a type error for now:

```luau
type Headbutt = "Hello"
type Name = string
-- TypeError: Interpolated string sub-expressions must be literal strings, but this expression is of type 'string'.
--    Type error span is here -> |    |
type PupilsDilated = `{Headbutt} {Name}`

-- We will allow boolean and nil singletons too. If number singleton types are ever introduced, supporting them should
-- be discussed seprately.
type PupilsConstricted = `{true} {nil}`
```

We also need to represent string singletons which depend on a generic type or unsolved type function - consider the
following code:

```luau
const function cat<Name>(person: Human<Name>): `Brush {Name}`
	return `Brush {person.name}`
end
```

We will create an unresolved type function to represent interpolated stringletons which cannot be resolved yet. This
type function won't be accessible in the user's environment. Analysis outputs containing these signatures will stringify
it syntactically to interpolation quotes and sub-expressions.

Currently, there are situations where a string expression will be inferred as a stringleton at a callsite. In any of
these, the callsite will contribute to inference of sub-expressions:

```luau
const function leopard<loaf>(value: loaf | keyof<some_tbl>): loaf
end

local trill = {
	chitter = ":3"
}
-- currently, this causes `foo.chitter` to infer to '":3"'
leopard(trill.chitter)
-- this currently instantiates leopard with 'loaf' as 'string'
-- instead, it could coerce foo.chitter.
leopard(`hello {trill.chitter}`)

const spit: unknown = ...
-- we will need to infer the lower bound of 'loaf' as 'string' here, because a sub-expression is not a singleton.
leopard(`hello {spit}`)
```

### Stringleton Unions as Sub-Expressions

Unions are particularly difficult to deal with. For now, we will define the behavior as shown in this code:

```luau
type Huff =
	| "Squeak"
	| "Click"
type NyaNya = "Purr"
-- This will materialize as "Hello Squeak Purr" | "Hello Click Purr"
type Kya = `Hello {Huff} {NyaNya}`
-- To avoid materializing exponentially many unions, this will emit a type error complaining about only up to one string
-- singleton union sub-expression being allowed.
type Mrrp = `{Huff} {Huff}`
```

## Drawbacks

- Backtick quotes will become unavailable in type syntax. It is likely any other usage for them would be confusing
anyways though.
- Increases the complexity of type inference around string singletons.

## Alternatives

- Do nothing. People can use existing workarounds for the forseeable future, and the syntactic consistency is not
critically important when weighed against potential drawbacks.
- Handle unions via a dedicated concrete type for interpolation. This approach is nonexclusive with this RFC, and can be
discussed at a later date.
- Introduce utility type function(s) based on string formatting
- Introduce concatenation syntax operator for types with `..`
