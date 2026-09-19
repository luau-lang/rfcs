# Scoped Module Imports for Luau  

## Summary  

Introduce a new syntax for scoped module imports in **Luau**, allowing selective imports of module members without requiring manual `local` assignments. This would streamline working with many dependencies, making code more readable and maintainable.  

## Motivation  

Currently, Luau requires modules to be imported using `require`, followed by manual assignments:  

```lua
local Pet = require(game.ReplicatedStorage.Library.Items.Pet)
local Currency = require(game.ReplicatedStorage.Library.Items.Currency)
```

In large Roblox projects, developers often need to require **20-50 individual modules** to accomplish a task. This results in repetitive and verbose module imports, making code harder to manage.  

I propose a cleaner syntax:  

```lua
from game.ReplicatedStorage.Library.Items require Pet, Currency
Pet("Dog")
Currency("Coins")
```

This improves readability and better reflects how modules are organized in large-scale Roblox development.  

### Expected Benefits  
- More concise and readable imports.  
- Eliminates redundant `local` assignments after requiring modules.  
- Reduces clutter in scripts that rely on many modules.  
- Better organization of large libraries with deeply nested module structures.  

## Design  

### Syntax  

```lua
from <ModulePath> require <Name1>, <Name2>
from <ModulePath> require <Name1> as <Alias1>, <Name2> as <Alias2>
from <ModulePath> require *
```

#### Example Usage  

```lua
-- Old way
local Pet = require(game.ReplicatedStorage.Library.Items.Pet)
local Currency = require(game.ReplicatedStorage.Library.Items.Currency)
local Egg = require(game.ReplicatedStorage.Library.Items.Egg)

-- New way
from game.ReplicatedStorage.Library.Items require Pet, Egg, Currency

-- Usage
Pet("Dog")
Inventory:Add(Currency("Diamonds"):SetAmount(500))
```

#### Import with Aliases  

```lua
from game.ReplicatedStorage.Library.Items require Pet as PetItem, Toy as ToyItem
PetItem("Dog")
ToyItem("Ball")
```

#### Import with Wildcard  

```lua
from game.ReplicatedStorage.Library.Items require *
Pet("Dog")
Toy("Ball")
```

### Semantics  

- `from <ModulePath> require <name>` is equivalent to `local <name> = require(<ModulePath>.<name>)`.  
- `from <ModulePath> require <name> as <alias>` binds the imported name to `<alias>`.  
- Multiple names can be imported in one statement.  
- The module is loaded only once, maintaining Luau’s existing `require` caching behavior.  

## Drawbacks  

- Introduces new syntax that deviates from Luau’s current module system.  
- Adds complexity to the compiler/parser.  
- May encourage excessive use of imports in a single script, potentially affecting readability.  

## Alternatives  

Create massive modules with dozens of requires built in, so called 'index' modules. This works, but floods the name space and stresses the type checking engine.
