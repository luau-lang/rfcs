# Documentation Comments

## Summary

Adds builtin support for documentation comments to luau.

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

## Design

### Whitespace Requirements

Documentation comments are to be automatically detected by luau, with them requiring no whitespace in between the end of the comment and whatever they're commenting on be it a variable, type, table, or function. Although a single new line is allowed at the end of a comment, as the following would be quite ugly:

```luau
--[[
	mrow meow mrrp
]]local function meow()
	-- code here
end
```

Whitespace sensitivity is also existent as to avoid issues with header comments becoming documentation comments as seen below.

```luau

--[[
	header
	I'm a header!
]]

type Meow = "mrrp" -- in luau lsp as of writing Meow has its documentation comment as the header comment
```

### Comment Requirements

Doc comments can be any comment, aslong as it follows the whitespace rules as defined previously.
There is a slight deviation here for better backwards compatiblity; where any extra dashes at the beginning will be removed from the comment when displayed, so for example this comment:

```luau
---- I have extra dashes!
```

Would appear as "I have extra dashes" instead of "-- I have extra dashes!".

### Carrying

Documentation comments automatically carry from variable to variable, and from type to type; unless overriden which can be seen in the example of 2 modules below below.

Module A:

```luau
-- mrrp
export type Meow = "mrrp"

type Mrrp = module_a.Meow -- Because Mrrp is just Meow, it'll have the same doc comment as Meow

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
Although fields in tables cannot have their comments overriden, with the comment thats used as a doc comment required to be directly above the first definition of the field:

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

### Type functions

Documentation Comments will also be exposed in type functions with the following methods added to the `type` instance:

#### `type` Instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `getcomment()` | `string?` | returns the comment attached to the type; if there is no comment attached it returns nil |
| `setcomment(comment: string?)` | `()` | adds / overrides the type's documentation comment; if comment is nil or is a string with a length of 0, removes the types's comment |

## Alternatives

Luau could require a specific type of comment for all documentation comments (not just short comments) like moonwave. But the goal of this RFC is not to have any fancy syntax for documentation comments, and instead have something that will work with the most amount of codebases today.
