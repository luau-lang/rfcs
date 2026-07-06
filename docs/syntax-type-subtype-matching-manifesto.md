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
- make cohesive type-level programming patterns more idiomatic to improve user experience and adoption within luau's new
type solver
- provide major performance benefits over traditional function overloads due to the linear priority and 

## design

this section is incomplete and serves as a rough draft of what the design might look like

some personal notes:
- i think we should implement this initially as syntactic sugar around type functions and subtyping, of course
respecting covariance. a built-in implementation could improve performance & inference a lot, which is a goal of
formalizing this, but it does not need to exist at first.
- runtime match expressions are intentionally out of scope for this rfc, and it seems like a bad choice to limit
ourselves to matches testable at runtime
- shortcircuiting should have major performance advantages over traditional overloading because the priority is clear,
and you don't need to check the validity of every arm.

rough syntax declaration:

```luau
-- i think we should probably require parenthesis on the pack here because it looks the most luau and also wont block
-- t {} type syntax from happening in the future
-- the first matching arm will be the output of this type function
type retrieve<entity, lifetime, dead> = match (lifetime, dead) {
	-- similarly to functions, parenthesis are required on the input pack but not the output pack.
	-- to avoid confusion with function types, ':' has been chosen as a separator between the input and output packs.
	-- we'll append an implicit ...any to the tail of this pack so it doesn't need to be written by hand
	("static"): index<entity, "data">,
	("dynamic", true): nil,
	-- trailing commas or semicolons are allowed
	-- '(): never' will be the final arm by default unless specified
	(): index<entity, "data">?,
}
```

closing questions (draft, temporary):
- if we implement this with current subtyping rules respecting covariance, users may shoot themselves in the foot with
inexact tables. this issue exists already when using the type system, but will likely surface more if we encourage users
to rely on subtyping. is there a more intuitive way to handle issues with covariance here?
- is there any other surprising (bad) behavior these packs will exhibit with existing subtyping rules?
- can type inference construct matches? if no, and if runtime matching were to ever exist, could type inference then
infer these types usefully?
- will this mesh with type pack unions?

## drawbacks

todo
- users may be encouraged to test subtyping against larger types when a simpler discriminant can be used trivially
- although the simplicity of making this sugard around user-defined type functions is advantageous, we should consider:
  - implementing it with a type function means non-unary return packs will need to be a type error until type pack
  functions are done
  - user-defined type functions might not have imperative access to match types
- todo...

## alternatives

- todo
