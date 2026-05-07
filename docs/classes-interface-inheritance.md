# Nominal Interface Inheritance for Classes

## Summary

```luau
-- All fields are public by default
interface Animal = { 
	species: string, 
	__tostring: (self) -> string
}

interface Reptile : Animal {
	species: string,
	__tostring: (self) -> string,
	sunbathe: (self) -> ()
}

class Cat : Animal
	public species: "Felis catus"
	public meowMult: number
	
	function __tostring(self): string
		return "I am a cat"
	end
	
	function meow(self): string
		return string.rep("meow\n", meowMult)
	end
end
```

## Motivation

Other languages with support for object-oriented programming support inheritance in some way. We (the Luau team) have even written libraries which would benefit from having classes with nominal inheritance. Interfaces also make it easier to write more modular code, and can provide another guidepost for programmers coming from other languages.

## Design

### Syntax
Interfaces are introduced with the new `interface` keyword. Like classes, they can only be declared at the topmost scope, and `export interface X = ...` is permitted.
Like classes, defining two classes with the same name in the same module is not allowed.

The right hand side of an interface declaration is almost syntactically identical to a table type, with the exception that there are no read/write access modifiers. Its contents are the fields and methods that the interface declares.

To declare that a class implements a particular interface, the class name is suffixed with `: <Interface name>` (see above for a concrete example of this). Interfaces can inherit from other interfaces as well via the same syntax. We do not consider multiple inheritance in this RFC, so attempting to define a class implementing multiple interfaces is a syntax error. Additionally, we consider only interface inheritance in this RFC, so `class Cls1 : Cls2` is also a syntax error.

Like classes, interface identifiers are hoisted to the top of the scripts in which they are defined.

### Runtime

#### Interface Objects

Evaluating an interface definition introduces an interface object in the module scope.

Interface objects are relatively uninteresting. The only operators they support are `==` and `~=`, and reflects physical (pointer) equality. `pairs`, `ipairs`, `getmetatable`, `setmetatable` will all error on interface objects, as will trying to iterate over them with a generic `for` loop, or trying to read or write a property of an interface. 

The `class` library's `isinstance` member will be extended to support interfaces:

```luau
interface Inter1 = {}

class.isinstance(Inter1, Inter1) -- true

interface Inter2 : Inter1 = {}

class.isinstance(Inter2, Inter1) -- true
class.isinstance(Inter1, Inter2) -- false

class Cl : Inter2 end

class.isinstance(Cl, Inter2) -- true
class.isinstance(Cl, Inter1) -- true
class.isinstance(Inter1, Cl) -- false
class.isinstance(Inter2, Cl) -- false

local inst = Cl {}

class.isinstance(inst, Inter2) -- true
class.isinstance(inst, Inter1) -- true
```

We introduce a new top type for interface objects: `interface`. The builtin `type()` and `typeof()` functions return `"interface"` for any class instance. For similar reasons to classes, we chose this over having them return the interface name (interface names do not have to be globally unique (they must only unique within a single module) and we do not want to make it possible for interfaces to impersonate other types).

```luau
interface Inter = {}

type(Inter) = "interface"
typeof(Inter) = "interface"
```

### Type System

Interface definitions introduce a new type to the type environment. Like classes, interfaces are nominally typed. We typecheck class and interface declarations that implement or extend interfaces according to regular subtyping rules.

For example:
```luau
type meat = ...
type plants = ...
type food = meat | plants

interface Animal = {
	species: string
	diet: food
}

interface Carnivore : Animal = {
	species: string
	diet: meat
}

class Lion : Carnivore
	species: "Panthera leo"
	diet: meat
end
```

`interface Carnivore : Animal` typechecks because `string <: string` and `meat <: food`. `class Lion : Carnivore` typechecks because `"Panthera leo" <: string` and `meat <: meat`.

Interfaces or classes with fields which are themselves interfaces or classes still follow nominal subtyping rules:

```luau
interface AnimalGroup = {
	member: Animal
}

class Pride : AnimalGroup
	member: Lion
end

class VenusFlyTrap
	species: "Dionaea muscipula"
	diet: meat
end

class MyGarden : AnimalGroup -- type error
	member: VenusFlyTrap
end
```

`class Pride : AnimalGroup` typechecks because `Lion` is nominally a subtype of `Animal` (`Lion <: Carnivore <: Animal)`. However, `class MyGarden : AnimalGroup` emits a type error since, even though `VenusFlyTrap` is a structurally a subtype of `Animal`, it does not nominally subtype `Animal`.

Note that method arguments are subtyped contravariantly:

```luau
interface Item = {}

class Treasure : Item end

interface TreasureChest = {
	store: (item: Treasure) -> ()
}

class MyTreasureChest : TreasureChest
	function store(item: Item)
	end
end
```

The above typechecks because `MyTreasureChest.store` has type `(Item) -> ()`. When we typecheck whether `MyTreasureChest` properly implements `TreasureChest`, we check whether `(Item) -> () <: (Treasure) -> ()`, which holds because function arguments are subtyped contravariantly and we have defined that `Treasure <: Item`.

#### The `self` type

Consider the above example again, this time with `store` defined as an instance method, rather than a static method.

```
interface Item = {}

class Treasure : Item end

interface TreasureChest = {
	store: (self: TreasureChest, item: Treasure) -> ()
}

class MyTreasureChest : TreasureChest
	function store(self: MyTreasureChest, item: Item)
	end
end
```

This time, the subtyping test for `store` goes as follows: 
- `(self: MyTreasureChest, item: Item) -> () <: (self: TreasureChest, item: Treasure) -> ()`
- `(self: TreasureChest, item: Treasure) <: (self: MyTreasureChest, item: Item)` (remember, we check method arguments contravariantly)
- `TreasureChest <: MyTreasureChest` This check fails because `MyTreasureChest` implements `TreasureChest`, not the other way around. 

We would like for interfaces to be able to specify instance methods as well. To this end, we introduce a new `self` type.

If the first argument of a class or interface method is unannotated and named `self`, it is judged to have type `self`. Subsequent method arguments can also be annotated `self`. `self` is also available for unnamed arguments of function types in interface definitions, as well as for return types. `self` has the same type as the interface or class in which it statically occurs.

To support typechecking classes and interfaces, `self` has the unique property that if it is present as a function argument, it is subtyped covariantly. So we can rewrite the above example as:

```luau
interface Item = {}

class Treasure : Item end

interface TreasureChest = {
	store: (self: self, item: Treasure) -> ()
}

class MyTreasureChest : TreasureChest
	function store(self: self, item: Item)
	end
end
```

And the subtyping test for `store` now passes:
- `(self: self, item: Item) -> () <: (self: self, item: Treasure) -> ()`
- `self: MyTreasureChest <: self: TreasureChest` (we replace each `self` with the class/interface in which they occur and perform the argument subtype test covariantly)
- `item: Treasure <: item: Item` (other arguments are still subtyped contravariantly)

For the purposes of this RFC, the new `self` type is defined only in the context of classes and interfaces, and that tables/metatables remain unaffected.

## Drawbacks

Again, there are quite a few moving parts here: this RFC is really two coupled features. Adding a `self` type doesn't make a ton of sense without inheritance, while interface inheritance doesn't work without `self`.

We introduce a new keyword, `interface`, and a new top type `interface`.

The `self` keyword is now doing double duty; it now has a meaning in type land as well as value land. 

## Alternatives

From the Classes RFC: 

> [Shared Self Types](https://github.com/luau-lang/rfcs/blob/master/docs/shared-self-types.md). This proposal was intended to shore up table type inference in the case that the code was written in an OO style, but after significant work, it doesn't actually work all that well in practice. The resulting system was very brittle and tricky to work with. Trickier, in fact, than the pattern that developers are already writing today.

We could've done implementation interface instead. Why not?

We aren't doing multiple inheritance here. Why not? Will we in the future?

Alternative syntax considered:
```
-- Interfaces don't need access modifiers on fields/methods
-- Function declaration style here diverges from how we do functions today (no closing end keyword)
interface Animal
	public species: string
	
	function __tostring(self): string
	-- alternatively
	public function __tostring(self): string
	-- or even
	public __tostring: (self: Animal) -> string
end
```