# User Defineable ``self`` Namecall Type

## Summary


The aim is to allow simple definition of ``self`` when defining functions with ``:`` in a class OOP context

```lua
function Class:TestMethod()
	self.@1
end
```


## Motivation

**Autocompletion is the motivation**.

See the "Alternatives" section to understand the motivation as well.

The point is to be able to define ``self`` in **namecall** defined functions as well.






## Design
Yet to be found.


## Drawbacks

Wouldn't see one for now.

## Alternatives

### Best Alternative

```lua
local Class = {}
Class.__index = Class


function Class.new()
	local obj = setmetatable({}, Class)
	
	obj.Value1 = 1
	obj.Value2 = 2
	
	return obj
end

type ClassObject = typeof(Class.new())


function Class:TestMethod()
	self = self :: ClassObject
	
	-- Now "self" has Value1 and Value2
end
```

This ``self = self :: ClassObject`` is the easiest way that I know about. Based on my analysis. The default compile optimization mode will not count that part into compilation.

Which is good, which means that this current way doesn't have any impact in anything. **But it is repetitive**.

However, one will be able to index ``Class:@1`` to **differentiate** methods defined by ``:``.



### Second Alternative
```lua
local Class = {}
Class.__index = Class


function Class.new()
	local obj = setmetatable({}, Class)
	
	obj.Value1 = 1
	obj.Value2 = 2
	
	return obj
end

type ClassObject = typeof(Class.new())


function Class.TestMethod(self: ClassObject)
	-- "self" has Value1 and Value2
end
```

**This is still repetitive**

The bigger issue is that this is not autocompletion friendly.

```lua
local Test = Class.new()

Test:@1
```
This will show it in the autocomplete. But that's not the issue.

The issue is when you use the pure Class table and index it with ``:``

```lua
local Class = {}
Class.__index = Class



function Class.new()
	local obj = setmetatable({}, Class)
	
	obj.Value1 = 1
	obj.Value2 = 2
	
	return obj
end

type ClassObject = typeof(Class.new())

function Class.TestMethod(self: (typeof(Class) & typeof(ClassObject)))
	-- "self" has Value1 and Value2
end

Class:@1
```

Notice how ``ClassObject`` is not resolved at that time.

If one wanted to do ``Class:@1`` to filter out all namecall methods in their code, without creating a constructor. They simply can't.

But this is a **metatable** in a class-construction context. Even Lua designs these OOP functions like that https://www.lua.org/pil/16.2.html



### Fixed type defined Alternative
This is an alternative that I captured off from **react-lua**.

```lua
local Class = {} :: {
  -- This is the thing that defines self
	[string]: (self: string) -> any
}
Class.__index = Class



function Class.new()
	local obj = setmetatable({}, Class)
	
	obj.Value1 = 1
	obj.Value2 = 2
	
	return obj
end

type ClassObject = typeof(Class.new())


function Class:TestMethod()
	self.@1
end
```

``self`` is now a ``string``. By defining ``[string]`` the automatic type resolver won't automatically put the actual things defined in the code into that table as well. That's why it isn't that good, but it's very smart. Hence why I called it "fixed" as in, _the definition is strictly already given._
