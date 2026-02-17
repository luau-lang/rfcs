# Type-only imports

## Summary

Add type-only importing to Luau, allowing code to import types without creating a runtime dependency.

## Motivation

Assume the creation of a `Computer` and `Mouse` class. Computer uses `Mouse` as a component, and `Keyboard` requires `Computer` as an argument. Typically, the first instinct is to use dependency injection:
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
Here, a cyclic dependency between `Computer` and `Mouse` makes retaining intellisense impossible. A popular workaround is introducing a third module solely for type definitions; essentially, a header file:
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
While this restores intellisense, it becomes cumbersome as more types and dependencies are added. The header file method works very well for types that can be created without the importing of external types, or put simply, a user-defined type that doesn't depend on other user-defined types.

An arguably more elegant solution is to allow code to directly import types, avoiding runtime dependencies:
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

Preferably, the design should adhere to the general appearance of Luau.
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
