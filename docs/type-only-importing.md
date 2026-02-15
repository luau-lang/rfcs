# Type-only imports

## Summary

Add type-only imports to Luau, allowing code to import types without creating a runtime dependency. This enables support for importing interfaces, type aliases, and dependency injection.

## Motivation

Assume the creation of a `Computer` and `Mouse` class. Computer uses `Mouse` as a component, and `Keyboard` requires `Computer` as an argument. Typically, the first instinct is to create something like:
```lua
-->> computer
local Mouse = require(script.Mouse)
local function createComputer()
  local self = setmetatable({}, Computer):: Computer
  self.Mouse = Mouse.new(self)
  return self
end
```
```lua
-->> mouse
local Computer = require(script.Parent)
local function createMouse(Computer: Computer.Computer)
  return setmetatable({Computer = Computer}, Mouse):: Mouse
end
```
However, it becomes clear that this is impossible, since this causes a cyclic dependency between `Computer` and `Mouse`. The logical next step would be to create a third container storing the `Computer` type, of which both `Computer` and `Mouse` require from it:
```lua
-->> computer
local Mouse = require(script.Mouse)
local __types = require(script.__types)
local function createComputer()
  local self = setmetatable({}, Computer):: __types.Computer
  self.Mouse = Mouse.new(self)
  return self
end
```
```lua
-->> mouse
local __types = require(script.__types)
local function createMouse(Computer: __types.Computer)
  return setmetatable({Computer = Computer}, Mouse):: __types.Mouse
end
```
This works, but can become very cumbersome very quickly, especially if `Computer` also depends on other user-generated types, which may or may not generate their own cyclic dependencies.
Reason being, the current type importing state of Luau is a clear limiting factor when considering pratices like dependency injection.

The above problem can easily be solved if code can directly import types, removing the need to depend on the module containing the types.
```lua
-->> computer
local Mouse = require(script.Mouse)
local function createComputer()
  local self = setmetatable({}, Computer):: Computer
  self.Mouse = Mouse.new(self)
  return self
end
```
```lua
-->> mouse
type Computer = typeget(script.Parent) --> no cylic dependency
local function createMouse(Computer: Computer) --> intellisense on "Computer" remains
  return setmetatable({Computer = Computer}, Mouse):: Mouse
end
```

## Design

Preferably, the desgin should adhere to the general appearance of Luau.
Here are two proposed syntaxes:
```lua
type Apple, Banana = typeget(script.Fruits)
type { Apple, Banana } = typeget(script.Fruits)
```
`type Apple, Banana = typeget(script.Fruits)` is more vanilla to Luau, but `type { Apple, Banana } = typeget(script.Fruits)` makes more semantic sense since type importing is unordered and named.

Optionally, inline type extraction:
```lua
type User = require("./UserRepository").User
```

This feature incidentlly also alleviates Luau's wierd `ModuleName.ModuleType` syntax.

Semantics:
- Type-only imports are erased at runtime (or used for optimization).
- No Lua code is emitted, so no runtime require happens.
- Regular type imports directly from other code continue to behave as usual.

## Drawbacks
More keywords. Possible confusion with type extraction via `require`. `require` essentially becomes a `typeget` and a code emitter combined.

## Alternatives
As aforementioned, an alternative could be a third, intermediary module. But as explained, could quickly become cumbersome.
