# Granular Imports

## Summary

Allow developers to directly import only necessary parts of libraries or modules as first-class variables.

## Motivation

- Less boilerplate for importing commonly used methods; repeatedly prefixing methods with module names (e.g. `Fusion.Computed()`, `Fusion.Observer()`) creates visual noise
- People who work with React, Fusion, and other components-based toolings
- Consistency with the incoming [`export`](https://github.com/luau-lang/rfcs/pull/42) keyword.
- Reinforced with relative imports
- Potentially enabling any optimization through static and read-only bindings

## Design

### Syntax

Given that this is the module utilizing incoming RFC features

```lua
export class Dog
  	private const Sound = "Bark";
  
	function New(sound: string): Animal
		local self = Animal();
		return self;
	end

	function Taunt(self: Animal): ()
		print(self.Sound);
	end
end
```

Selectively import objects, methods, and other values from a module

```luau
import { Dog } from "./Animals";
```

Selectively import objects, methods, and other values from a module under different names

```luau
import { Dog as dog } from "./Animals";
```

Federalize the import

```luau
import * as Animals from "./Animals";
const Dog = Animals.Dog;
```

Granular imports applied in practice

```luau
import { Computed, Observer, Tween, Spring } from "@ReplicatedStorage/Packages/Fusion";
```

## Drawbacks

- Lua already has `require()` that can be done in one line. However, it is not too ergonomic, and it can cause visual strain with repeated mentions of the module name or the keyword.
- This is a huge change that can go against Lua's design philosophy. Though, we must weigh utiliarian and identity.
- Adding a new import keyword alongside `require()` may cause inconsistencies in collaborative environments.

## Alternatives

## "Lua" approach

In a more unique Lua approach while maintaining less boilerplate without the keyword, we can use the syntax:

```
const { Dog } = require("./Animals");
```

Though, this implies a limited opportunity for developers to be able to rename values to their namespace needs.
