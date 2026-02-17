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
@[deprecated { use = "dog"}]
local puppy = "whimper"

return table.freeze({
	puppy = puppy,
	dog = "woof",
})
```

Although this will cause the return in the module to be linted, but this lint won't be passed on to consumers of the module:

Module:

```luau
-- DeprecatedApi: Variable 'puppy' is deprecated, use 'dog' instead Luau(22)
return table.freeze({
	puppy = puppy,
	dog = "woof",
})
```

Consumer:

```luau
-- No lint occurs on the import
local module = require("@module")

-- DeprecatedApi: Member 'puppy' is deprecated, use 'dog' instead Luau(22)
print(module.puppy)
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

If the value of a field has attributes, those will be merged with the attributes defined on the field.
With the attributes on the field having priority over the attributes on the value.
For example: if both the value and the field have a `@deprecated` attribute, the `@deprecated` attribute on the value will be ignored.
With the `@deprecated` attribute on the field being used instead.

```luau
@[deprecated { reason = "cat is a more modern API"}]
local function get_cat_sound()
	return "meow"
end

-- DeprecatedApi: Function 'get_cat_sound' is deprecated, cat is a more modern API Luau(22)
local bad_module = table.freeze({
	get_cat_sound = get_cat_sound,
})

-- No lint occurs, because the @deprecated attribute of the 'get_cat_sound' function has been overridden
-- by the @deprecated attribute of the field 'get_cat_sound'.
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
@deprecated
type Puppy = "whimper"

-- DeprecatedApi: Type 'Puppy' is deprecated Luau(22)
type CanineSounds = {
	puppy: Puppy,
	dog: "woof",
}

-- No lint occurs, because the @deprecated attribute of the type 'Puppy' has been overridden
-- by the @deprecated attribute of the field 'puppy'.
type PetSounds = {
	-- Just like with tables, entries have their attributes merged with the values attributes.
	@deprecated{ use = "dog" } puppy: Puppy,
	dog: "bark",
	cat: "mrrp",
}
```

Although type declarations can also have attributes directly after the `=`,

## Drawbacks

Allowing attributes to be on types, variables, and table fields, would be added complexity to the language. Although would be more inline with what a user would expect/want, as its odd from the perspective of a user that currently types can't be marked as deprecated.
