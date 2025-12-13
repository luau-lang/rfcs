# Extendable Returns for Tables with an Unique Reference

## Summary

When a function creates a new table _which proves to be of a fresh and sole unique reference_, we should be able to safely assume that it can be returned and assigned
to a variable like usual, _**but also able to extended it further**_. Just as if you'd have declared a table normally or used ``table.clone``
and then assigned further properties to _widen_ the table.


## Motivation

We can declare a table and assign it to a variable and extend it further, like so:
```lua
--!strict
local tbl = {}
tbl.prop = "hello"
```
We can declare a table, and widen it further with any properties we desire.

&nbsp;

But the Type Checker doesn't let us widen tables that a function created and returned.
```lua
--!strict
function makeTable()
  return {}
end

local tbl = makeTable()
tbl.prop = "hello" -- type error
```

Because of this:
- You can't have pre-filled tables or templates, without fully annotating everything you'd ever put into them.
  - And you get type warnings if you attempt to do this.

<br>

But what if you could do it?



## Design

Any function that creates a table in itself as a standalone, can be proven to be of a sole unique reference. And therefore we can say that this table can be extended
further when it is returned.

So as a quick overview for an unique reference:
- The function creates a table by itself, e.g. ``{}``
- The Type Checker can guarantee that this is the sole reference of the table within the function.
  - Which makes it safe to widen the table with further properties when returned.
 


```lua
--!strict
function makeTable()
  local tbl = {}
  tbl.assignMeLater = nil :: number?
  tbl.entry = 1
  return tbl
end

local tbl = makeTable()
print(tbl.assignMelater, tbl.entry)
tbl.prop = "hello" -- Does NOT Type Error!
```

It would be as if you declared all the table fields without using any functions.
```lua
local tbl = {}
tbl.assignMeLater = nil :: number?
tbl.entry = 1
tbl.prop = "hello"
```

<br>

The table that the function returns, would be an unique reference per call. 
```lua
--!strict
function makeTable() return {} end
-- Imagine the below but if as if you'd have written
-- local tbl_1 = {} instead

local tbl_1 = makeTable()
local tbl_2 = makeTable()

tbl_1.ONE = 1

-- tbl_2 would not contain "ONE" (unless someone would write tbl_2.ONE = 1, of course)
tbl_2.TWO = 2
```


<br>

From where the function comes from is irrelevant, aslong the table the function creates, is proven to be of an unique reference.


```lua
--!strict
function makeTable()
	return { one = 1 }
end

function makeTable2()
	local tbl = {}
	tbl.secondTbl = makeTable()
	return tbl
end

local tbl = makeTable2()
tbl.secondTbl.two = 2 -- works
```

<br>

This would also allow you to use table templates.

```lua
local Helper = {}

function Helper.createTable()
	local tbl = {}
	tbl.Init = nil :: () -> ()?
  return tbl
end

return Helper
```

You'd get hinted at, that you have an ``.Init`` function
```lua
--!strict
local Helper = require("./Helper")
local module = Helper.createTable()

function module.Init() end
function module.DoSomethingElse() end -- No Type Error, and works

return module
```

As a result this ``module`` would be extendable and you'd also have ``.DoSomethingElse`` included into it, thanks to Type Inference.
And ``typeof`` would resolve the type as usual, and when resolving that function type, the expected behavior would be, that you get a unique reference
based on that table it created.



## Drawbacks

- If ``table.clone`` ever gets a feature where the reference of a table type is cloned. What would its type annotation look like?


Currently, any function written without explicit annotations would be impacted.
```lua
function createStuff() return {x=0,y=0,z=0} end
```

```lua
--!strict
local myStuff = createStuff()
myStuff.oops = 5 -- This wouldn't type error anymore
```

You'd have to do
```lua
function createStuff(): {x: number, y: number, z: number} return {x=0,y=0,z=0} end
```

And then you'd get the warning at ``myStuff.oops``. But this would mean that you'd have to manually annotate it.
There should be a way to automatically say, that the table strictly isn't meant to be _widenable_.


## Alternatives

Without this RFC, the only alternative is to annotate everything ahead,
or say ``{[string]: any}`` but that would sewer you away from property names and autocomplete.
