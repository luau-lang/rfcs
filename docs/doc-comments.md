# Documentation Comments

## Summary

Adds builtin support for documentation comments to luau.

## Motivation

Currently it is up to each language server to decide how to parse documentation comments,
leading to inconsistancies such as the following:

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

Documentation comments are to be automaticaly detected by luau, with them requiring no whitespace inbetween the end of the comment and whatever they're commenting on be it a variable, type, table, or function.
Although a single new line is allowed at the end of a comment, as the following would be quite ugly:

```luau
--[[
	mrow meow mrrp
]]local function meow()

end
```

Whitespace sensitivity is also existant as to avoid issues with header comments becoming documentation comments as seen below.

```luau

--[[
	header
	I'm a header!
]]

type Meow = "mrrp" -- in luau lsp as of writing Meow has its documentation comment as the header comment
```
</br>

Doc comments also require comments to be long comments:
```luau
--[[
	I'm a long comment!
]]

--[=[
	I'm also a long comment!
]=]
```
Or short comments with an extra dash `---`:
```luau
--- I'm a short documentation comment!
```

Short comments are otherwise disallowed, as they are generally just notes like `--TODO: meow`
or `--the solver will kick and scream if this doesnt exist`.
</br>

Documentation comments are able to be detected for any of the following cases seen in the codeblocks below:
```luau
-- BaseNode is a simplified version of a type I've ripped from a library of mine
type BaseNode<HOS, S, N> = {
	--[[
		Indicates if the node has only a single supporter, this is purely internal
		and only used by `object_tree.closest_empty_node`,
		as an optimization for trees that have a taper type of "Flat" or "Slope".
	]]
	has_one_supporter: HOS,
	--[[
		Array of nodes that are supported by this node, if `nil` this node is at the bottom of the tree.
	]]
	supported: { N }?, -- not using the node type here because then the recursive type error comes to haunt me
	--[[
		Indicates if this node is taken,
	]]
	taken: boolean,
	--[[
		The CFrame this node is at within the DataModel
	]]
	cframe: CFrame,
	--[[
		Either an array of nodes that are supported by this node, or if `has_one_supporter` is true this points to a node directly below this node,
		and if `nil` this node is at the top of the tree.
	]]
	supporter: S?,
}

-- Node if used will have documentation comments (this example is here because it once didn't work in luau lsp)
export type Node = BaseNode<true, Node, Node> | BaseNode<false, { Node }, Node>
```

```luau
--- Indicates if the plugin is enabled
local ENABLED = true

--- Sets the Enabled state of the plugin
local function SET_ENABLED(s: boolean)
	ENABLED = s
end
```

```luau
local module = {
	--- meow
	meow = "mrrp"
}

--- This documentation comment will not override the documentation comment for module.meow
module.meow = true
```

```luau
local module = {}

--[[
	This documentation comment will show! Because despite not being defined in the brackets,
	its the first place where meow is defined
]]
module.meow = true
```

```luau
local module = require("@module")

-- This type will have the same documentation comment as 'Meow' type from the module
export type Meow = module.Meow

--[[
	This documentation comment overrides the 'Mrrp' type from the module
]]
export type Mrrp = module.Mrrp

-- The same goes for functions/variables returned by the module!
return {
	mrow = module.mrow,
	--- I override module.maow's documentation comment!
	maow = module.maow,
}
```

## Alternatives

Luau could require a specific type of comment for all documentation comments (not just short comments) like moonwave. But the goal of this RFC is not to have any fancy syntax for documentation comments, and instead have something that will work with the most amount of codebases today.
