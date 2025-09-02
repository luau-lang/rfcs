# Documentation Comments

## Summary

Adds builtin basic support for documentation comments to luau, without any specifications as to what documentation comments must have in their body.

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

Documentation comments are to be detected by luau as any comment that has can have an optional new line before a `type`, `variable`, `function` parameter, or `table` property declaration.

```luau
--TODO: Not a documentation comment

-- im a documentation comment :3c
type Meow = "mrrp"
```

Having a whitespace requirement ensures comments are intended to be documentation comments and not any other type of comment, like a header comment:

```luau
--!native

--[[
	url
	utility functions for encoding and decoding urls
]]

export type SpaceEncoding = "+" | "%20" -- SpaceEncoding won't have the header as its documentation comment
```

Theres also an array of ignored prefixes for documentation comments that can be defined in `luaurc` files.
This is further explained under the [Configuration](#configuration) section.

### Function Parameters

Function parameter are an exception to the whitespace requirements, this is so luau doesn't have to learn if the user is using tabs as spaces, or just tabs.

```luau
local function make_them_meow(
	-- The name of the person who will meow
	person: string,
	-- The length of the meow
	len: number?
)
	print(`{person} meowed: me{string.rep("o", len or 1)}w`)
end
```

Although luau will take note of the whitespace before the comment, so long comments like this:

```luau
local function make_them_meow(
	--[[
		The name of the person
		who will meow
			Note: cannot be "tim"
	]]
	person: string
```

Will display like this:

```plaintext
The name of the person
who will meow
	Note: cannot be "tim"
```

Instead of:

```plaintext
	The name of the person
	who will meow
		Note: cannot be "tim"
```

### Trimming

#### Extra whitespace

Extra whitespace is to be trimmed from the start and end of comments, as is done today by luau-lsp and Roblox:

```luau
--[[
	I have a tab!
]]
```

Would appear as `I have a tab!` instead of `​ ​ ​ ​ ​ I have a tab!`.

#### Extra dashes

Extra dashes are also trimmed on documentation comments for better backwards compatiblity with luau-lsp:

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

Documentation comments automatically carry from variable to variable, and from type to type; unless overriden which can be seen in the example of 2 modules below below.
This mimics luau-lsp's existing behavior for handling documentation comments.

Module A:

```luau
-- mrrp
export type Meow = "mrrp"

type Mrrp = module_a.Meow -- Because Mrrp is just Meow, it'll have the same documentation comment as Meow

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
Although fields in tables cannot have their comments overriden:

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

### Configuration

A new key `documentation` will be added to [`luaurc`](./config-luaurc.md) files for conifguring how documentation comments will work, with the following options being underneath it:

#### `requirements`

A map containing optional requirements that can be enabled that effect how documentation comments are detected.

* `allowEndComments`

	Allows comments at the end of things to become their documentation comments:

	```luau
	local function meow(
		f: () -> (), -- I'm the documentation comment!
	)
		-- code here
	end
	```

	Although with this setting enabled luau will still prefer comments above parameters,
	and will only use end comments if there is no comment above.
	Luau will also not allow newlines between end comments and whatever they're documenting, as to avoid the following:

	```luau
	local function meow(
		f: () -> (),
		-- I'm the documentation comment for f!
		mrrp: "meow"
	)
		-- code here
	end
	```

* `allowMultilineShortComments`

	Allows multiline short comments to be viewed as documentation comments, as normally the following would only have the last short comment displayed:

	```luau
	-- I'm not displayed by default!
	-- I'm displayed by default!
	local ENABLED = false
	```

#### `ignoredPrefixes`

An array of prefixes that indicate any comments starting with these prefixes are to not be detected as documentation comments.
For example if someone wanted to not have `--TODO:` comments detected, they would define the following:

```json
{
	"documentation": {
		"ignoredPrefixes": [ "TODO:" ]
	}
}
```

**Note:** Ignored prefixes are applied after [trimming](#trimming).

### Type functions

Documentation Comments will also be exposed in type functions with the following methods added to the `type` instance:

#### `type` Instance

| Instance Methods | Return Type | Description |
| ------------- | ------------- | ------------- |
| `documentation()` | `string?` | returns the documentation attached to the type; if there is no documentation attached it returns nil |
| `setdocumentation(documentation: string?)` | `()` | adds / overrides the type's documentation; if documentation is nil or is a string with a length of 0, removes the types's documentation |

For table fields their documentation will be attached to the key type instances, and function parameters will have their documentation be attached to the type instances that make up the head and tail.

## Alternatives

Luau could require a specific type of comment for documentation comments like moonwave. But the goal of this RFC is not to have any fancy syntax for documentation comments, and instead have something that will work with the most amount of codebases today.
