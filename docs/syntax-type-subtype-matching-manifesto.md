# introduce type subtype matching syntax

## unfinalized

this rfc is a draft not intended to be merged as-is. after getting feedback on initial ideas,
design / drawbacks / alternatives will be refined & completed fully.

## summary

introduces syntax to construct a type function which branches on subtyping of a type pack, acting as a strong
alternative to traditional methods of overloading type signatures.

## motivation

any first-class support for programmatic concepts (such as control flow) in the type system carry similar benefits to
those of [user-defined type functions](https://rfcs.luau.org/user-defined-type-functions.html) in precision, synthetic
types, and user experience. this rfc aims to:
- use simple syntax to complement existing traits found in user defined type functions
- enable an expeditious implementation timeline with good room for improvement from type pack functions & native support
- make type-level programming patterns more idiomatic to improve user experience and adoption within luau's new
type solver
- use match syntax to improve verifiability and inference of programs with complex type expression
- provide major editor performance benefits over traditional function overloads and type functions

## design

this section is incomplete and serves as a rough draft of what the design might look like

some personal notes:
- i think we should implement this initially as syntactic sugar around type functions and subtyping, of course
respecting variance. a built-in implementation could improve performance & inference a lot, which is a goal of
formalizing this, but it does not need to exist at first.
- runtime match expressions are intentionally out of scope for this rfc, and it seems like a bad choice to limit
ourselves to matches testable at runtime

rough syntax declaration:

```luau
-- a very simple example
-- the first matching arm will be the output of this function
local function meow<bool>(): match (bool) {
	(true): some_value,
	(false): nil,
	(boolean): some_value?,
	(): never, -- this is the default final arm, defined explicitly for this example
}
end
-- i think we should probably require parenthesis on the pack here because it looks the most luau and also wont block
-- t {} type syntax from happening in the future
type retrieve<entity, lifetime, dead> = match (lifetime, dead) {
	-- similarly to functions, parenthesis are required on the input pack but not the output pack.
	-- to avoid confusion with function types, ':' has been chosen as a separator between the input and output packs.
	-- we'll append an implicit ...any to the tail of this pack so it doesn't need to be written by hand
	("static"): index<entity, "data">,
	("dynamic", true): nil,
	-- trailing commas or semicolons are allowed
	(): index<entity, "data">?,
}
```

closing questions (draft, temporary):
- is there any other surprising (bad) behavior these packs will exhibit with existing subtyping rules?
- can type inference construct matches? if no, and if runtime matching were to ever exist, could type inference then
infer these types usefully?
  - the answer is, in narrow circumstances, almost certainly:
  ```luau
  local result: match (
  	typeof(foo)
  ) {
  	(~false?): typeof(bar),
  	(false?): typeof(baz),
  	(): typeof(bar) | typeof(baz),
  }
  ```
- could the type system infer generic bounds from match syntax?
  - probably yes, any arm which returns `never`(s) could constrain input types
  - there may be branches where you want to explicitly emit a user-defined type error. Should we
  add discrete syntax to surface specific errors and 'explanations' from matches?
- can the type system verify returns with match types?
  - it seems likely it could be far superior to existing function overloads. i'm currently unsure how much flexibility
  there is with the soundness and completeness of that tho
- will this mesh with type pack unions?

## drawbacks

todo
- users may be encouraged to test subtyping against larger types when a simpler discriminant can be used trivially
- users might expect covariance when their match expressions search for e.g. `{ unknown }`. subtyping issues like this exist today, but will surface more if we encourage logic based on subtyping
- although the simplicity of making this sugar around user-defined type functions is advantageous, we should consider:
  - implementing it with a type function means non-unary return packs will need to be a type error until type pack
  functions are done
  - user-defined type functions might not have imperative access to match types
- todo...

## alternatives

- todo
