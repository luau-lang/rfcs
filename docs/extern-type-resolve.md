# Resolve extern types by name in type functions

## Summary

Add `types.extern(name: string): type` to the type-function library. This function resolves a globally registered extern type by its name and returns the corresponding type value, preserving its nominal identity. This allows user-defined type functions to turn computed names or string singleton type arguments into references to host-provided types.

## Motivation

Type functions can inspect and transform types passed to them, but cannot currently look up an arbitrary extern type using a computed string. For instance: `types.singleton("Instance")` cannot be resolved to a type of `Instance`.

This distinction matters for host environments with large sets of extern types, such as Roblox classes. A type function that receives a class name as a string singleton should be able to return the corresponding class type without maintaining a manually written mapping of every supported name.

The expected outcome is a small lookup API that supports these mappings while retaining the identity and relationships of the original extern types.

This would be especially useful in Roblox, where APIs such as `Instance.new`, `Instance:IsA(className: string)`, `Instance:FindFirstChildWhichIsA(className: string)` and `Instance:QueryDescendants(query: string)` use strings to identify classes, with specialized type-checking behavior implemented as magic functions in C++. Resolving these names to extern types would allow user-defined type functions to express similar relationships and derive new types from the properties of named classes.

## Design

The type-function library gains the following function:

```luau
types.extern(name: string): type
```

`name` is an ordinary string within the type-function runtime. Lookup is case-sensitive and searches the root global type scope for a binding whose underlying type is an extern type. The binding must not require generic type or type-pack arguments.

The returned type represents the existing extern type rather than a newly constructed structural equivalent. Its nominal identity, properties, and inheritance relationships are preserved.

Names may also be computed inside a type function before being passed to `types.extern`.

```luau
type function resolveExtern(name: type): type
    assert(name:is("singleton"), "Expected a string singleton")

    local text = name:value()
    assert(typeof(text) == "string", "Expected a string singleton")

    return types.extern(text)
end

type ResolvedInstance = resolveExtern<"Instance">
-- ResolvedInstance refers to the same extern type as Instance.
```

In Roblox's own API, the `Instance.new` function could be implemented as a type function through the following strategy:

```luau
type function CreateInstance(ty: type)
	if not ty:is("singleton") then
		print("Expected a singleton, got:", ty.tag)
		return nil
	end

	local name = ty:value()
	
	if type(name) ~= "string" then
		print("Expected a string, got:", name)
	end
	
	local success, extern: type = pcall(function ()
		return types.extern(name)
	end)
	
	if not success then
		print("Could not find extern type:", name)
		return nil
	end
	
	local root: type? = extern
	
	while root do
		if tostring(root) == "Instance" then
			return extern
		end
		
		root = root:readparent()
	end
	
	print(`Extern type: {name} is not an Instance.`)
end

local NewInstance: <ClassName>(
	className: ClassName & "",
	parent: Instance?
) -> CreateInstance<ClassName> = Instance.new

-- Resolved type for part would be: `Part`
local part = NewInstance("Part") 
```

This proposal deliberately excludes:

- Looking up local aliases, imported aliases, or bindings in child scopes.
- Resolving primitive types such as `"number"`; existing fields such as `types.number` remain available.
- Parsing type expressions such as `"Instance?"`.
- Instantiating generic aliases.
- Creating new extern types or looking up ordinary runtime values.

Local aliases do not shadow this lookup. For example, a local `type Instance = number` does not change the result of `types.extern("Instance")`.

## Drawbacks

* String-based lookup is less discoverable and less refactor-friendly than a direct type reference. Misspellings are diagnosed when the type function is evaluated rather than through ordinary name resolution.
* Singleton strings are still finicky to interact with in certain use cases, requiring generics unioned with `""` to forcefully narrow the type down to a singleton string, which might be confusing.
* Results depend on the extern types registered by the host. A type function that resolves a Roblox-specific class will not necessarily work in another embedding environment.
* Resolving an extern type incurs serialization work, subject to the same complexity limits as other type-function operations.

## Alternatives

- **Pass the type directly.** A type function can receive `Instance` as a type argument instead of receiving `"Instance"`. This is preferable when the type is known statically, but does not support names computed within a type function.
- **Maintain an explicit mapping.** Users can write a table or conditional mapping names to available type values. This avoids a new API but requires maintenance as the host's extern types change.
- **Provide an extern lookup table** A global table (e.g. `types.externs`) mapping extern names to their singleton types.

Without this feature, users must pass types directly or maintain their own name-to-type mappings.

## Prior Art

Luau's existing `types` library provides primitive type values, constructors such as `types.singleton`, and operations for inspecting types. `types.extern` complements these APIs by retrieving an existing named extern type rather than constructing a new type.

TypeScript can express a related mapping using an explicit interface and indexed access types, for example `Registry[Name]`. That approach requires a declared registry; this proposal instead uses the extern bindings already supplied by the Luau host.

Runtime reflection APIs such as .NET's `Type.GetType` also retrieve type representations using strings.
