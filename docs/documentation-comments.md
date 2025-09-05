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

* Short and long comments can't be merged together, as they're distinctly different types of comments.
* Comments before a declaration are favored over ones after. So if a declaration has comments before and after, only the comment before will be used.
* Multiline short documentation comments don't work after a declaration, as they require newlines in-between the short comments they're made up of.

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
	-- A wedge shape with a slope on one side.
	| "Wedge"
	-- A wedge shape with slopes on two sides.
	| "CornerWedge"

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

Theres also an array of ignored prefixes for documentation comments that can be defined in [`luaurc`](./config-luaurc.md) files.
This is further explained under the [Configuration](#configuration) section.

### Trimming

#### Whitespace

Whitespace is to be trimmed from the start and end of each line in comments, as is done today by luau-lsp and Roblox:

```luau
--[[
	I have a tab!
	I also have a tab!
]]
```

Appearing as:

```plaintext
I have a tab! I also have a tab!
```

Although empty lines are still preserved, but multiple empty lines in a row will be reduced to just one empty line:

```luau
--[[
	meow


	mrrp
]]
```

Appearing as:

```plaintext
meow

mrrp
```

#### Outer dashes

Extra dashes are also trimmed on documentation comments for better backwards compatibility with luau-lsp:

```luau
--- I have an extra dash!
```

Would appear as `I have an extra dash!` instead of `- I have an extra dash!`.
Dashes are also removed at the end as to support this common styling:

```luau
--[[
	I have extra dashes at the end!
--]]
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
}

-- Module B: for all your needs of bees and cats
local export = {
	buzz = "mrrp",
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

If a field isn't defined in a table literal, fields will only allow their comment to be the first place where the field is defined:

```luau
local module = {}

do
	-- The callback for this module
	module.callback = function() end
	-- I'm not the documentation comment!
	module.meow = "mrrp"
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

### Configuration

A new key `documentation` will be added to [`luaurc`](./config-luaurc.md) files for configuring how documentation comments will work, with the following options being underneath it:

#### `ignoredPrefixes`

Array of prefixes that indicate any comments starting with these prefixes are to not be detected as documentation comments.
With ignored prefixes being applied after [trimming](#trimming), so there is no need to allow for string patterns.
For example if someone wanted to not have `--TODO:` comments detected, they would define the following:

```json
{
	"documentation": {
		"ignoredPrefixes": [ "TODO:" ]
	}
}
```

Multiline short comments are treated as separate comments, up until one of the short comments doesn't start with an ignored prefix:

```luau
--TODO: add types
--TODO: add code
-- im a multiline
-- short documentation comment :3
local function meow()
	--code here
end
```

With the example displayed as:

```plaintext
im a multiline
short documentation comment :3
```

</br>

Prefixes also get excessive whitespace at the start and end trimmed off by luau (with luau providing a lint telling the user to remove said excessive whitespace if [Support Luau-syntax configuration files](https://github.com/luau-lang/rfcs/pull/125) is merged).
The reasoning for having excessive whitespace trimmed is because of comments like this:

```luau
  --ROBLOX TODO: useTransition,
  --ROBLOX TODO: startTransition,
  -- ROBLOX TODO: useDeferredValue,
  -- ROBLOX TODO: REACT_SUSPENSE_LIST_TYPE as SuspenseList,
  unstable_LegacyHidden = ReactSymbols.REACT_LEGACY_HIDDEN_TYPE,
```

Also because theres probably someone out there that does this:

```luau
--[[
	TODO: I'm a long todo!! SO SO SO long that the user decided I couldn't fit in a short comment!!!
]]
```

#### `inferModuleHeaderAsDocumentationForReturn`

Boolean indicating if header comments in files should be used as documentation for the return of modules, with a default of false.
This option also trims the first line of the header if it has the same text as the file name.
Header comments are detected by them being at the top of the file after hot comments and whitespace, with them not being detected as being attached to anything that can have documentation.

```luau
--!native

--[[
	stack_tree
	library for generating tree structures for objects that are stacked
	to prevent floating objects from ever happening
]]

--[[
	stack_tree has its documentation as:
	library for generating tree structures for objects that are stacked
	to prevent floating objects from ever happening
]]
local stack_tree = {}

return stack_tree
```

### Type functions

Documentation Comments will also be exposed in type functions with the following methods added to the `type` instance:

#### `type` Instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `documentation()` | `string?` | returns the documentation attached to the type; if there is no documentation attached it returns nil |
| `setdocumentation(documentation: string?)` | `()` | adds / overrides the type's documentation; if documentation is nil or is a string with a length of 0, removes the types's documentation |

For table fields their documentation will be attached to the key and value type instances, and function parameters will have their documentation be attached to the type instances that make up the head and tail.

**Note:** Primitive type instances on the `types` library (`types.number`, etc) are not allowed to have their documentation set, unless cloned using `types.copy`.

## Alternatives

Luau could require a specific type of comment for documentation comments like Moonwave. But the goal of this RFC is not to have any fancy syntax for documentation comments, and instead have something that will work with the most amount of codebases today.

## Future Work

Develop an alternative to hot comments, so the documentation luaurc keys can be exposed nicely per file.
As this the same type of bad as const in lua:

```luau
--!documentation ignoredPrefixes "TODO:"
```
