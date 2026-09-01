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

## Design
We allow function generics to transfer over the underlying singleton rather than its broader type to allow further analysis via type functions.

This will make function generics transfer (if possible) underlying singletons directly. 
The circumstances under which this is possible depend entirely on the generic parameter value, specifically if it can be a singleton or not.

## Drawbacks
**Complexity** Allowing generics to transfer singletons can complicate analysis in the type functions, as compared to simple tags like `string`,`boolean`,`nil` and `number`
analysis would have to be done via checking the actual value via `singletontype:value()`

**Compatibility** Existing type function code that relies on transferring of the upper type rather than the singleton will break

## Alternatives
**Improve existing api** Adding similar value function to `string`,`number`,`nil` and `boolean` types that if available returns the actual value of the type can allow
the same exact behavior

