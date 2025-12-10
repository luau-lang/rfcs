# Allow attributes to be used on types, variables, and table fields

## Summary

Allows [Attributes](./syntax-attributes-functions.md) to be applied to types, variables, and table fields in Luau.

## Motivation

Currently attributes can only be applied to functions, this means types cannot be marked as [`@deprecated`](./syntax-attribute-functions-deprecated.md).
Nor can variables or table fields be marked as [`@deprecated`](./syntax-attribute-functions-deprecated.md).

## Design

Note: With this RFC the internals for attributes will be able to state they can be applied to only certain bits of syntax, so one can't apply [`@native`](./syntax-attribute-functions-native.md) to a type or variable and have it work. Although this RFC does not state how this would be done in luaus internals.

The following list proposes how attributes should be attached to each bit of syntax in luau (except functions and comments):

### Variables

This exists as it could be useful for if for instance a constant is defined as a local variable and then exported:

```luau
@[deprecated { reason = "dog is a more modern API"}]
local puppy = "whimper"

return table.freeze({
	puppy = puppy,
	dog = "woof",
})
```

### Tables

```luau
local pet_sounds = {
	@deprecated cat = "meow"
}

@deprecated module.dog = "woof"

@deprecated
module.parrot = "cracker, now"

return table.freeze(pet_sounds)
```

Note: If the value of a entry has attributes, those will be merged with the attributes defined on the entry.
With the attributes on the entry having priority over the attributes on the value.

```luau
@[deprecated { reason = "cat is a more modern API"}]
local function get_cat_sound()
	return "meow"
end

local module = table.freeze({
	@[deprecated{ use = "cat" }] get_cat_sound = get_cat_sound,
	cat = "meow"
})

-- DeprecatedApi: Member 'get_cat_sound' is deprecated, use 'cat' instead Luau(22)
module.get_cat_sound()
```

### Types

Types work similarly to [Variables](#variables) and [Tables](#tables), except being types.

```luau
@[deprecated{ use = "PetSounds" }]
type pet_sounds = {
	parrot: "cracker, now",
	dog: "woof",
	cat: "meow",
}

@deprecated
type Puppy = "whimper"

type PetSounds = {
	-- Just like with tables, entries have their attributes merged with the values attributes.
	@deprecated{ use = "dog" } puppy: Puppy,
	dog: "bark",
	cat: "mrrp",
}
```

## Drawbacks

Allowing attributes to be on types, variables, and table fields, would be added complexity to the language. Although would be more inline with what a user would expect/want, as its odd from the perspective of a user that currently types can't be marked as deprecated.
