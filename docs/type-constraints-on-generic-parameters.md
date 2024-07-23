# Type Constraints on Generic Functions

## Summary

Modify the current generics implementation to allow types to be constrained.

## Motivation

When writing generic functions currently, it is as if you were simply using any unless you make your own type, being that the generic type has no constraints.

Take this snippet for example:
```luau
type Gun = {
    IsEmpty: (self: Gun) -> boolean,
    FireGun: (self: Gun) -> (),
    -- ...
}

local function fireGun<T>(self: T)
    if not self:IsEmpty() then -- Type Error: Type 'T' does not have key 'IsEmpty'.
        self:FireGun()         -- Type Error: Type 'T' does not have key 'FireGun'.
    end
end 

local gun: Gun

fireGun(gun)             -- Works as "intended".
fireGun("Hello, world!") -- Will error you out, but it will compile as if it were valid.

```
On this snippet, we are trying to implement a theoretical Gun system, we may have many implementations for the Gun, like a Shotgun or a Pistol, and we may want to use generics to implement them, however, the current system for generics does not allow for this without getting type errors.

This is the main reasoning behind this modification. Many other languages have a feature to limit what types does a generic allow as a baseline. Languages like C# have this implemented on the form of the `where T : BaseClass` syntax, which delimits the type that `T` can be or inherit from. It supports cases in which you may want to use a strucutre, but not know its implementation details, like an interface.

## Design

The design of this feature will reuse the already existing generics syntax, with little modification.

```luau
type Gun = {
    IsEmpty: (self: Gun) -> boolean,
    FireGun: (self: Gun) -> (),
    -- ...
}

type Shotgun = {
    -- ...
} & Gun

local function fireGun<T: Gun>(self: T)
    if not self:IsEmpty() then -- No error
        self:FireGun()         -- No error
    end
end 

local gun: Gun
local shotgun: Shotgun

fireGun(gun)                 -- No error
fireGun(shotgun)             -- No error

fireGun("Hello, world!")     -- Type Error: type 'string' and 'Gun' are not related to one another.
```

The syntax for defining a generic function would remain the same, except, instead of simply taking a `T`, we would take a `T : TypeConstraint`, the syntax remains similar to other features of the language, and it would work by checking if the type that `T` is inferred to be has any kind of relationship with `TypeConstraint`

This allows us to give functionality using generics reliably and safely without having to define our own types to enforce anything of any kind, resulting on improvements on the overall developer experience, as we could get autocompletion for when a variable is related to `TypeConstraint`, but abstract it accordingly. More over, it could allow for improvements on Native Code Generation, by allowing the generation of independant functions for each type the generic has been used for.

## Drawbacks

The main deterrant to implementing type constraints on generics is type intersection. Type intesection allows us to implement this snippet without fighting much of anything.

```luau
type Gun = {
	IsEmpty: (self: Gun) -> boolean,
	FireGun: (self: Gun) -> (),
	-- ...
}

type Shotgun = {
	-- ...
} & Gun

local function fireGun(self: Gun)
	if not self:IsEmpty() then
		self:FireGun()
	end
end

local gun: Gun = nil
local shotgun: Shotgun = nil

fireGun(gun) 		            -- Works as intended.
fireGun(shotgun) 	          -- Works as intended.
fireGun("Hello, world!")    -- Type Error: type 'string' could not be converted into 'Gun'

```

Using type intersection would not require any changes to generics, which undermines the point that generics were made to get across, which is one function that can take many inputs, be type safe and all of it without having to make a new function that does the same on a type that has a similar structure.


## Alternatives

The unspoken of alternative to this, is simple type intersection, yet another reason as to why this should be implemented is that the future of `Luau` is bleeding more into a safer language in what types respect to, by adding this feature we are improving an existing one which was added with such idea in mind, improving it by adding something that was originally not thought of and that could overall improve the current implementation for generics, and allow for more flexible patterns when programming with Luau.
