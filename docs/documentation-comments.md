# Documentation Comments

## Summary

Adds builtin basic support for documentation comments to luau, without any specifications as to what format documentation comments must have their body be.

## Motivation

Currently it is up to each language server to decide how to parse documentation comments, leading to inconsistencies such as the following:

```luau
--- miaow
type Meow = "mrrp"

local x: Meow -- "miaow" will be shown as a documentation comment with luau lsp, but not in studio
```

Documentation comments also get stripped from types when a type is put into a type function, as language servers have no way of detecting that the documentation comment should still be there.

```luau
type function Meow(t)
	return t
end

--[[
	I'm a documentation comment!
]]
type Mrrp = "miaow"

type Mrow = Meow<Mrrp> -- no documentation comment
```

## Comment Terminology

This RFC refers to 2 types of comments, short and long. Theres also a third type of comment called a "Multiline short comment", but it's really just multiple short comments merged together.

```luau
--[[
	I'm a long comment!
]]

-- I'm a short comment!

-- I'm a multiline
-- short comment!
```

## Design

Documentation comments are to be detected by luau as any comment or short comments that is before (with one newline allowed), or after (with no newlines allowed) a `type`, `variable`, `function` parameter, `type` union component, or `table` property declaration.
There are some caveats to this though:

* Only short comments can be merged together, as short comments merging with long comments goes against how one would think documentation comments should work. The same goes for merging long comments with other long comments.

	```luau
	--[[
		It'd be very weird
	]]
	--[[
		if
	]]
	-- we merged together
	```

* Comments before a declaration are favored over ones after. So if a declaration has comments before and after, only the comment before will be used.
* Multiline short documentation comments don't work after a declaration, as they require newlines in-between the short comments they're made up of.

Header comments are an exception where despite not being directly over what they're documenting they can be inferred as the documentation for a modules return. If the header comment meets the following requirements:

* The first line matches the file name, with whitespace at the start and end of the first line not included.
	Spaces are allowed in-between words for improved legibility if the file name is in `camelCase`, `PascalCase`, `snake_case`, or `kebab-case`. So if your file name is `meowMrrp` you can have the first line of the header as `meow mrrp`, `Meow Mrrp`, `meow Mrrp`, `Meow mrrp`, or `meowMrrp`.
* The modules return doesn't have a documentation comment already
* Nothing but hot comments (`--!strict`), or whitespace is above it

</br>

```luau
-- PartType example taken from: https://create.roblox.com/docs/en-us/reference/engine/enums/PartType

-- The PartType enum controls the Shape of a Part.
type PartType =
	-- A spherical shape.
	| "Ball"
	-- A block shape.
	| "Block"
	-- A cylinder shape oriented along the X axis.
	| "Cylinder"
	| "Wedge" --[[ A wedge shape with a slope on one side. ]]
	| "CornerWedge" -- A wedge shape with slopes on two sides.

--[[
	I'm not included in the documentation because I'm long and not short!
]]
-- im a multiline
-- short documentation comment :3
type Mrow = "meow"

-- I get ignored because short and long comments can't be combined!
--[[
	I'm the documentation!
]]
type Mrrp = "maow"

-- im used :3
type Meow = "mrrp" -- im not used :(
```

</br>

Having newline requirements ensures comments are intended to be documentation comments and not any other type of comment, like a header comment:

```luau
--!native

--[[
	url
	utility functions for encoding and decoding urls
]]

export type SpaceEncoding = "+" | "%20"
```

### Carrying

Documentation comments automatically carry from variable to variable, and from type to type; unless overridden which can be seen in the example of 2 modules below below.

Module A:

```luau
-- mrrp
export type Meow = "mrrp"

type Mrrp = Meow -- Because Mrrp is just Meow, it'll have the same documentation comment as Meow

-- I override Meow's documentation comment!
type Mrow = Meow

return nil
```

Module B:

```luau
local module = require("@module")

-- buzz buzz
type Export = {
	-- The comment of buzz is me instead of module_a.Meow's comment!
	buzz: module_a.Meow,
	mrow: module_a.Meow, -- The field 'mrow' is just module_a.Meow, so it'll have the same documentation comment as module_a.Meow
}

-- Module B: for all your needs of bees and cats
local export = {
	buzz = "mrrp",
	mrow = "mrrp",
}

-- Casting here overrides the type & comments of export to be that of the type 'Export'
return export :: Export
```

</br>

Although fields in tables cannot have their comments overridden:

```luau
local module = {
	-- The callback for this module
	callback = function() end,
}

-- I meow, but I cannot override!
module.callback = function()
	print("meow")
end
```

Except for table literals, but they'll lint:

```luau
local module = {
	-- The callback for this module
	callback = function() end,
	-- I meow! But I get linted :(
	callback = function()
		print("meow")
	end,
}
```

</br>

If a field isn't defined in a table literal, fields will only allow their comment to be the first place lexically where the field is defined:

```luau
local module = {}

if true then
	-- The callback for this module
	module.callback = function() end
else
	-- I'm not the documentation comment!
	module.callback = "mrrp"
end
```

Although this ignores functions that aren't called by the module itself before it returns, so the following isn't allowed:

```luau
local module = {}

function module.meow()
	-- I'm the documentation comment!
	module.callback = function() end
end

return module
```

Example of a function called by the module itself before it returns:

```luau
local module = {}

pcall(function()
	-- The callback for this module
	module.callback = function() end
end)

return module
```

### Type functions

Documentation Comments will also be exposed in type functions with the `:setdocumentation()` method on type instances. It should be noted that this method will not work on primitive type instances on the `types` library (ie types.number, etc), unless copied using `types.copy`.

| Overload | Return Type | Description |
| ------------- | ------------- | ------------- |
| `(documentation: string?)` | `()` | adds / overrides the type's documentation; if documentation is nil or is a string with a length of 0, removes the types's documentation |
| `(copyfrom: type)` | `()` | sets the documentation of the type to be same as the provided type's documentation |

</br>

Table fields will have their documentation attached to the key type instances, with values having a different documentation comment. So when hovering over the value of the field `sound`, a language server can show the documentation for `PurrMeow`:

```luau
-- a combination between a purr and a meow
type PurrMeow = "mrrp"

type CatInfo = {
	-- the sound the cat is currently making
	sound: PurrMeow
}
```

Function parameters will have their documentation be attached to the type instances that make up the head and tail.
Although if [Function Parameter Names in User-Defined Type Functions](<https://github.com/luau-lang/rfcs/pull/137>) is accepted, function parameters will work like how table fields do. With the parameters documentation attached to the name, and the values having their own documentation.
So when hovering over the value of the parameter `sound`, a language server can show the documentation for `PurrMeow`:

```luau
local function set_sound(
	-- The cat that will have its sound changed
	cat_info: CatInfo,
	-- The new sound to set the cat to make
	sound: PurrMeow
)
	cat_info.sound = sound
end
```

## Alternatives

Luau could require a specific type of comment for documentation comments like Moonwave. But the goal of this RFC is not to have any fancy syntax for documentation comments, and instead have something that will work with the most amount of codebases today.
