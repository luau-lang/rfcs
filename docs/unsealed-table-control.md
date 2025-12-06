# Table Sealing and Unsealing Control

## Summary

This proposes a way to control Sealing and Unsealing in a lightweight way, to allow the user to alter and aid ``--!strict`` mode and the type checker.
Especially, the sealing constraints for tables and or adapting from Old Solver to New Solver.


## Motivation

The Old Solver allowed things that the New Solver does not allow or _"support"_.

**Old Solver notes:** _(These are just mentioned to reflect upon the Old Solver, and the past bugged things)_

<details>
  <summary>Old Solver - Leaked Free Tables (Bad)</summary>

The Old Solver mistakenly allowed something that was called _"Free Table State"_, visualized as ``{- -}``.
A Free Table was an internal table state, which acted as a _per-file permanent_ unsealed table, which was **broken type**.
It seemed like it was used to show you what properties you can input into a table, while initializing it for a function argument.

There were glitches that allowed you to have a free table.
```lua
function createStarterTable<free>(input)
	input.StarterValue1 = 1
	
	return input :: typeof(input) & free
end


local tbl = createStarterTable({})
tbl.ABCDE = 2 -- free table addition
tbl.ACE -- As you type, the entry would get added to the type and Autocomplete will suggest this, because the Free Table type is broken. This is a bug.
-- The autocomplete would suggest read and write properties, that's why it thinks "ACE" would have been part of the properties.
```

Alternative:
```lua
function createStarterTable(input)
	input.StarterValue1 = 1

	return input :: typeof(input) & typeof(table.freeze())
end

local tbl = createStarterTable({})
tbl.ABCDE = 2 -- free table addition
tbl.ACE
```

</details>
<details>
  <summary>Old Solver - Unsealed Table tricks</summary>

In some cases ``typeof({})`` would work.


```lua
--!strict
function modify(a)
	a.O = "e"
	return a
end

-- local tbl = {} -- <-- has to be made here
-- this here doesn't work if the line with "Note1" is present

local tbl = setmetatable({}, {}) 
tbl.Original = 1 -- Note1: This here would cause an error at "modify(tbl)" without setmetatable

tbl = modify(tbl)

tbl.ABCDE = "e"
```

Alternatively, ``local tbl = ({} :: any) :: typeof(setmetatable({}, {}))``

Though this would cause it to now be a metatable.


Another way would be
```lua
--!strict
function modify(tbl)
	tbl.A = 1
	return tbl -- You don't have to return, but the autocomplete works better like that
end

--local tbl = {}
local tbl = {}
tbl = modify((tbl :: any)) -- "any" to silence the Type Error

tbl.moreStuff = "test"
```

</details>

&nbsp;

&nbsp;

----

Are there cases where ``--!strict`` feels too "strict"? There aren't really ways to _tweak_ such behavior,
other than changing to _completely_ change to other modes.

```lua
--!strict
function modify(input)
  -- input infers as "input: { A: number }"
  -- this causes it to expect "A" to already be present
	input.A = 1
	return input -- You don't have to return, BUT it helps the autocomplete somehow
end

local tbl = {}
tbl = modify(tbl) -- Type Error: Type 'tbl' could not be converted into '{ A: number }'

-- Type Error: Cannot add property 'moreStuff' to table '{ A: number }'
tbl.moreStuff = "test"
```


**Because of this, the user has to:**
- Use type annotation as a **boilerplate**.
  - This forces the user to declare a field e.g. ``.moreStuff`` in **two** places, which is difficult to maintain.
<details>
  <summary>Type Annotation Boilerplate - Case 1</summary>
This case here creates a table. But it becomes sealed once it it leaves ``prepare()``

```lua
--!strict
local function prepare()
	local t = {}
	t.Value1 = nil :: string?
	return t
end

local tbl = prepare()
-- Value1 now had to be written in two places at once
-- The only difference is that the definition lives in prepare(), which is more portable.
tbl.Value1 = "a"

tbl.B = 1 -- Type Error, you have to add "B" at the top as well...
```

</details>
<details>
  <summary>Type Annotation Boilerplate - Case 2</summary>

```lua

```

</details>

&nbsp;
- The user has to abandon ``--!strict``.
- Refactor code, just to avoid simple initializers.

&nbsp;

&nbsp;

See the following:

```lua
--!strict
local tbl = {}
tbl.StarterValue1 = 1
tbl.moreStuff = 2
```

```lua
--!strict
local tbl = {}
do
  tbl.StarterValue1 = 1
end
tbl.moreStuff = 2
```

And now see

```lua
--!strict
function modify(input)
	input.StarterValue1 = 1
	return input
end

local tbl = {} -- <-- table was created in this scope
tbl = modify(tbl)

tbl.moreStuff = 2 -- Type Error due to sealing
```

All just sounds the same.
It's not just about the type checking anymore, it's also about the autocomplete that started getting influenced by the New Solver.


Not only that, but if you have a pre-set table template, and you distribute the code,
others would also have to go to wherever the template is, to modify it.

## Design

The goal here is to allow the user to tweak the ``--!strict`` behavior, through lighter annotation.

For instance, that function pattern above where a table is mutated from an upvalue scope, should at least be allowed.

However, implementation wise, it might be too difficult to automatically determine whether or not a table should return back unsealed.
Hence **why the idea** is to let the user annotate this.

- This allows the user to tweak the strictness behavior
- You can already silence warnings using tricks like ``(nil :: any)``.
  - The type solver doesn't complain about it. If we allow to silence issues,
    then it is arguably not fair if there wouldn't be additional ways to do similar, or alter the behavior.
  - The user **awarely** types it in.


**Example Designs:**
It would have to be a mixture of a generic type, or telling that a generic type, is a mutable type.

Or essentially, we're marking a table to not return back sealed when it exits the scope it was marked as "to unseal".

**To clarify:**
- Unsealing a table is not permanent
- The table will seal again if not marked.
- The table should not repeat the _"Leaked Free Table State"_ mistake from the Old Solver
- A module's direct returned table should stay sealed.


```lua
function modify(input: {...})
	input.StarterValue1 = 1
	return input
end

local tbl = {}
tbl = modify(tbl)

tbl2.moreStuff = 2 -- This is fine
```

If we could say that ``input: {...}`` is unsealed, we would expect that ``modify`` returns the contents back unsealed.

However, the table will **only** return unsealed for this operation. If anything else is not annotated to expect to treat it "unsealed",
then it will seal it again, just like it would behave by default.

For instance:

```lua
function modify2(input)
	input.StarterValue2 = 2
	return input
end

tbl = modify2(tbl) -- <-- this is sealed
tbl.moreStuffAgain = 2 -- Type Error
```
Now, the above would already type error at ``modify2(tbl)``, so now the problem is, can we mutate, but also not return as unsealed?

&nbsp;

Whether the syntax should be ``{...}`` or ``@unsealed`` is another question to ask.

Doing ``function modify(input: {...})`` is straightforward, and doesn't need extra annotations.

Extra annotations like:
```lua
local function unsealed(): unsealed<{ foo: number }>
    return { foo = 1 }
end
```
If it would be done like this, we would have 2 fields that have to be maintained again.



&nbsp;

Second idea, table types that are permitted to be extended.
```lua
type basic_template = {
	Name: string, -- This says that Name MUST be present
	... -- This tells it that more properties can be added into it
}
```

Question to ask, is when should ``.Name`` be present?

<details>
  <summary>Pseudo</summary>

```lua
--!strict

type basic_template = {
	Name: string,
  ...
}

local item = {
	expectItToContain<basic_template>
}
--item.Name = "Item"


-- We never satisfied "item" ?
return item
```

</details>

&nbsp;

Another question to ask is about ``typeof({})``, all this is, is a trick.
```lua
-- Assuming that SharedTable.new() gives you a table
local foo = SharedTable.new() :: typeof({})
foo.bar = 2
```
Without ``typeof({})`` there would be nothing.

But how official is ``typeof({})`` ?

Would this make more sense?
``SharedTable.new() :: {...}``


&nbsp;


Another thing to ask is what is allowed to be unsealed?

```lua
function Floor1()
	
	local function Floor2()
		
		local function Floor3(input)
			input.Hello = 1
			
			-- As "Unsealed"
			return input
		end
		
		local tbl = {} -- we created the table here
		tbl = Floor3(tbl)
		
		tbl.Hello2 = 2		
		
		-- As "Unsealed"
		return tbl
	end
	
	
	local catchedTbl = Floor2()
	
	-- Should "catchedTbl" be unsealed?
	

	print(catchedTbl)
end

Floor1()
```

``local tbl = {}`` was made in ``Floor2``, and modified by ``Floor3``. Then ``Floor2`` modifies it again.

Should ``Floor1`` ever be allowed to have that table as Unsealed? Or should ``Floor1`` initialize the table first instead?
They can still pass it through sub-scopes.

Now, incase ``Floor1`` wouldn't be allowed, it would mostly mean that tables can't get unsealed if they exit their definition scope.
And maybe that would be a little bit too restrictive as well.


## Drawbacks

Wrong usage would mean that the user runs into errors that could have been hinted by type checking.
But this is already possible regardless, e.g. with ``any`` and etc.

But wrong usage would mean that you'd get undesirable field additions. But then again, compare ``any``.

Would using ``{...}`` take a way to ever have "spread operation" from JavaScript?


## Alternatives
- The Old Solver has "workarounds"... But that's also the Old Solver.
- See the "Type Annotation Boilerplate" examples.
