# Allow generics to transfer singletons

## Summary
Generic functions will be able to transfer singletons directly

## Motivation

As of writing function generics only transfer the type of parameter rather than the underlying singleton we directly provided it with.

Currently the code below raises an error:
```luau
type function checkSingleton(a:type)
	if a.tag ~= "singleton" then
		error("Singleton expected, got: "..a.tag)
	end
	
	return a
end

local function refine<T>(a:T) : checkSingleton<T>
	return a
end

type singletonString = checkSingleton<"Singleton string"> -- ok
local a:singletonString = refine("Singleton string") -- not ok, error "Singleton expected, got: string"
```

This limits the usage of function generics for type functions due to automatic widening of the singleton via the generic. 
Some common string ordering operations have to be implemented on the developer side rather than api side.

As an example:

```luau

type ServiceTable = {
  GoodService:{};
}

local service_table:ServiceTable = {
  GoodService = {};
}

local function GetService(service:string) : {}
  --we ignore the internal type errors here as they do not serve the purpose of explaining the current motivation
  if not service_table[service] then
    service_table[service] = {}
  end
  return service_table[service]
end

local unknownservice = GetService("GoodService") -- ok but untyped

local goodservice:index<ServiceTable,"GoodService"> = GetService("GoodService") -- the variable is now typed but requires boilerplate
```

Currently existing enviroments (such as Roblox) have to cheat and introduce third-party code to make this analysis on the api end. This is unwanted as we want to make the typing features operate standalone without relying on outside code:

- `ServiceProvider:GetService()`
- `Instance:FindFirstChild()`
- `Instance:WaitForChild()`

This proposal can open up a possibility to make these functions entirely rely on built-in type functions via feeding them the singleton via the methods proposed below:

## Design
We allow function generics to transfer over the underlying singleton rather than its widened type to allow further analysis.

We propose 2 ways to implement the new behavior

### New default behavior
We introduce a new rule for the keyword `const`, where now it can also be put before a type to explicitly widen it to the original type. So `const "A string"` resolves to `string`

Under this behavior we disable automatic type widening similar to typescript, this does break existing code pieces like:

```luau
local t = {}
table.insert(t,"string") --ok
table.insert(t,"string2") --not ok, "string2" is not a "string"
```

The fix would be to change the underlying table.insert function type to, as a proposal:

```luau
type insert = <T>(t:{const T},insert:T) -> ()
```

this makes it so when we do feed a singleton to the function, the table type refines it to the widened type instead of the singleton, making the code above work properly:

```luau
local t = {}
insert(t,"Another string")
insert(t,"Singleton string") -- ok, table takes in a string, not the "Singleton string"
```

### Explicit behavior

Under this behavior existing automatic type widening remains. So we instead introduce a new token `!` which can be put before a type to disable automatic widening. As an example:

```luau
type function analyzestring(t:type)
	if t.tag == "singleton" then
		if type(t:value()) == "string" then
			for i,v in string.split(t:value(),",") do
				print(v)
			end
		else
			error("Not a string singleton")
		end
	else
		error("Not a singleton")
	end

	return t 
end

local function Test1<T>(a:T) : analyzestring<T> return a end
local function Test2<T>(a:T) : analyzestring<!T> return a end

Test1("Cool,string") -- not ok, error "Not a singleton"
Test2("foo,bar") -- ok, hint appears "foo","bar" but no errors
```

## Drawbacks
**Complexity** Allowing generics to transfer singletons can complicate analysis in the type functions if it would be implemented as the new default behavior, as compared to the simple tags like `string`,`boolean`,`nil` and `number` analysis would have to be done via checking the actual value via `singletontype:value()`

**Syntax additions** Depending on the implementation we would have to introduce a new token to analyze for, which complicates the language and introduce additional type analyzing time.

**Compatibility** Existing type function code that relies on transferring of the upper type rather than the singleton will break in the new default behavior

## Alternatives
**Improve existing api** Adding similar value function to `string`,`number`,`nil` and `boolean` types that if available returns the actual value of the type can allow the same exact behavior

