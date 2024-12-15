# `result` type function

## Summary

This RFC proposes the addition of a new type function `result`, which can be used to make result types in luau. That are consistent with the idiom established by the [pcall and xpcall global functions](https://luau.org/library#global-functions) in the language currently.

## Motivation

Currently, in order to properly type out a result one would have to write the following:

```luau
type Mrow = ((meow: string, mrrp: number) -> (true, string)) &
	((meow: string, mrrp: number) -> (false, nil))

--[[
	Note: the function is being cast to any, 
	as this rfc is being written with the assumption the developer is using --!strict mode
--]]
local mrow: Mrow = (function(meow: string, mrrp: string)
	if math.random() > .5 then
		return true, `{meow} {mrrp}`
	else
		return false, nil
	end
end) :: any
```

Due to to that making the developer have to write an overloaded function, and with overloaded functions not consistently working. Most will instead do this:

```luau
type Mrow = (meow: string, mrrp: number) -> (boolean, string?)
```

Leading to having the type burden passed onto the developer using the function, as now they have to cast the result if they want to avoid type errors:

```luau
local success, result = mrow("cat food", ":3")

if success then
	local new_result: string = result :: any
	print(`new food of type: {new_result}!`)
else
	error("no food :(")
end	
```

Some may take a diffrent approach by making their own result type, where they break from the luau idiom:

```luau
type Result<S, F = nil> = {
	ok: true,
	value: S
} | {
	ok: false,
	value: F
}

local function mrow(meow: string, mrrp: string): Result<string>
	if math.random() > .5 then
		return { ok = true, value = `{meow} {mrrp}` }
	else
		return { ok = false, value = nil }
	end
end
```

## Design

The functionality of the `result` type function is special, with it being an exception as it'll create a type-pack union. Thus using the result type function will not require the developer to write an overloaded function type.

```luau
type result<S..., F...>
```

```luau
local function mrow(meow: string, mrrp: string): result<string>
	if math.random() > .5 then
		return true, `{meow} {mrrp}` -- wont error

		return true, nil -- Error message: type 'nil' is not of type 'string'
	else
		return false, nil
	end
end
```

## Drawbacks

This type function will need to be special cased, complicating maintenance for the type solver. But, having a result type would allow for pcall and xpcall to get proper types in the future. Where calls to `error` within a function being pcalled would set the second typepack in the result type to the type of the value `error` was called with. 

```luau
local function mrow(meow: string, mrrp: string): string
	if math.random() > .5 then
		return `{meow} {mrrp}` 
	else
		error("no cats allowed")
	end
end

-- inferred as result<string, string>
local success, result = pcall(mrow, "cat food", ":3")
```

## Alternatives

Allow for type-pack unions to be written by developers, with the syntax probably looking like this:
```luau
local function mrow(meow: string, mrrp: string): (true, string) | (false, nil)
	-- code here
end
```
With this example breaking backwards compadibility with some types developers may have written already, as today the following is allowed:
```luau
-- inferred as: ((meow: string, mrrp: string) -> (true, string)) | false 
type mrrp = (meow: string, mrrp: string) -> (true, string) | false
```
Although it should be mentioned that result types are the only valid usecase for type-pack unions, and just having a result type instead of general type-pack union syntax would remove a potential footgun. Of someone using type pack unions for something thats not a result type. 

Do nothing, and leave it up to developers if they want to write overloaded functions, or make their own result type.
