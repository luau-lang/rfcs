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

This is counter-intuitive and limits the usage of function generics for type functions. 
Some common string ordering operations aren't possible with the current behavior and currently require separate typing.

As an example:

```luau

type ServiceTable = {
  GoodService:{};
}

local service_table:ServiceTable = {
  GoodService = {};
}

local function GetService<T>(service:T) : index<ServiceTable,T>
  --we ignore the internal type errors here as they do not serve the purpose of explaining the current motivation
  if not service_table[service] then
    service_table[service] = {}
  end
  return service_table[service]
end

local errorservice = GetService("GoodService") -- type error "Property 'string' does not exist on type '{ GoodService: Workspace }'"

local goodservice:index<ServiceTable,"GoodService"> = GetService("GoodService") -- the variable is now typed but the function still type errors
```

Currently exsiting enviroments (such as Roblox) have to cheat and introduce third-party code to make this analysis. This is unwanted as we want to make the typing features operate standalone without relying on outside code:

- `ServiceProvider:GetService()`
- `Instance:FindFirstChild()`
- `Instance:WaitForChild()`

This proposal can open up a possibility to make these functions entirely rely on built-in type functions via feeding them the singleton (such as with index<>)

## Design
We allow function generics to transfer over the underlying singleton rather than its broader type to allow further analysis via type functions.

This will make function generics transfer (if possible) underlying singletons directly. 
The circumstances under which this is possible depend entirely on the generic parameter value, specifically if it can be a singleton or not.

We also introduce a new built-in type function called `broad` which accepts any type and outputs stripped of their singleton type, for example the result of `broad<"Singleton string">` would actually be a `string`, not the singleton we fed it. The reason it accepts any type and not just singletons is that there are circumstances where we want to refine generics while accepting other non-singleton types such as in table.insert where we technically allow any type not just singletons.

The way this can be done is by either making this behavior the new default or by making it explicit.

### New default behavior
Under this behavior we make generics transfer the default behavior, this does break existing code pieces like:

```luau
local t:{string} = {}
table.insert(t,"string") --not ok, "string" is a singleton
```

The fix would be to change the underlying table.insert function type to as an example:

```luau
type insert = <T>(t:{broad<T>},insert:T) -> ()
```

this makes it so when we do feed a singleton to the function, the table type refines it to the it's broader type instead of singleton, making the code above work properly:

```luau
local t:{string} = {}
insert(t,"Singleton string") -- ok, table takes in string, not the "Singleton string"
```

### Explicit behavior

Under this behavior existing generic type transfer remains. Instead we introduce a new token `!` which can be put before a type to tell the type system to feed the underlying singleton directly. As an example:

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

this does mean that internally the system would still have to feed singletons. But it will choose to instead feed the broad type unless explicitly told by the `!` token to feed the singleton directly.

## Drawbacks
**Complexity** Allowing generics to transfer singletons can complicate analysis in the type functions if it would be implemented as the new default behavior, as compared to simple tags like `string`,`boolean`,`nil` and `number` analysis would have to be done via checking the actual value via `singletontype:value()`
**Syntax additions** Depending on the implementation we would have to introduce a new token to analyze for, which complicates the language and introduce additional type analyzing time.
**Compatibility** Existing type function code that relies on transferring of the upper type rather than the singleton will break in the new default behavior

## Alternatives
**Improve existing api** Adding similar value function to `string`,`number`,`nil` and `boolean` types that if available returns the actual value of the type can allow the same exact behavior

