# Classes

## Summary

```luau
class Point  
	public x: number  
	public y: number  
	private cachedLength: number?  
  
	constructor(x: number, y: number)  
		self.x = x  
		self.y = y  
	end  
  
	function length(): number  
		if not self.cachedLength then  
			self.cachedLength = math.sqrt(self.x * self.x + self.y * self.y)  
		end  
	  
		return self.cachedLength  
	end  
  
	function __add(other: Point): Point  
		return Point(self.x + other.x, self.y + other.y)  
	end  
  
	function __tostring(): string  
		return `Point \{ x = {self.x}, y = {self.y} \}`  
	end  
end  
  
local p = Point(3, 4)  
print(p:length()) -- 5.0 
```
This RFC proposes native class syntax with:
- `public` & `private` fields
- constructors
- implicit `self` for instance methods
- nominal typing
- native class instances
- supported metamethods


## Motivation

Many Luau users write object-oriented code today.  It should be a first-class, polished feature.

Accurate type inference of `setmetatable` has proven to be very difficult to get right. Because of this, the quality of autocomplete and type inference for object-oriented code is not as effective as it could be. A native class construct provides explicit semantics that are significantly easier for both the compiler and type checker to reason about. This allows class types to be nominal, instance layouts to be known statically, and member access to be analyzed more accurately.

Likewise, a construct with a fixed shape and a completely locked-down metatable will open up optimization opportunities that could improve performance:
- If classes can only be declared at the top scope, then we know that each method of each class has exactly one instance. This makes it simple for the compiler to know the exact function that will be invoked for any method call expression.
- If a value is known to be an instance of a particular class, the bytecode compiler should be able optimize method calls to skip the whole  `__index`  metamethod process and instead generate code to directly call the correct method.
- By the same token, method calls can be inlined more aggressively, such as self-method calls like  `self:SomeOtherMethod()`.
-   Field accesses can compile to a simple integral table offset so that the VM doesn't need to do a hashtable lookup as the program runs.
- Instance layouts can be split from field metadata to improve cache locality.

However, a class abstraction is more than a fixed-layout record with attached methods. Constructors and encapsulation are fundamental components of a class abstraction and establish the model developers interact with daily. Designing these semantics alongside the core class syntax results in a more cohesive feature than introducing them in separate RFCs after the surface language has already been established.

## Design

### Syntax

Class definitions are a block construct. `export class X` is allowed. 

Fields are introduced with the new `public` and `private` keywords. Members are public only when explicitly marked `public`.

```luau
class Player
    public name: string
    private coins: number = 0
end
```
A class may declare:
 - fields
 - one constructor
 - methods
 - supported metamethods

Classes are closed declarations. Fields, constructors, methods, and metamethods may only be declared within the lexical body of the class. Instance methods are invoked using `instance:method()` syntax. References to instance methods may be obtained through `Class.method` syntax.

```luau
local fn = Player.method
fn(player)
```

Instance methods and constructors implicitly receive a local binding named `self`, whose type is the enclosing class. This binding is available within constructors, methods, and metamethods.

Defining two classes with the same name in the same module is forbidden.

Classes can define the following Luau metamethods.  They all work just like they do on a metatable:

* `__call`
* `__concat`
* `__unm`
* `__add`
* `__sub`
* `__mul`
* `__div`
* `__mod`
* `__pow`
* `__tostring`
* `__eq`
* `__lt`
* `__le`
* `__iter`
* `__len`
* `__idiv`

For forward-compatibility, it is a syntax error to define any other method whose
name starts with two underscores.

#### Constructors
Classes may define a constructor using the contextual keyword `constructor`.
```luau
class Player
    public name: string

    constructor(name: string)
        self.name = name
    end
end
```
Instances are constructed by calling the class.
```luau
local p = Player("Builderman")
```
Additional rules include:
- Only one constructor is allowed per class.
- Constructors do not return values.
- Constructors cannot be invoked directly.
- Default field initializers execute before the constructor body.
- If no constructor is declared, a default zero-argument constructor is synthesized. Field default initializers still run. 
- Fields without default initializers are initialized to `nil`. 
- A field whose declared type does not include `nil` must be definitely assigned by either a field initializer or every control-flow path through the constructor. Failing to do so is a type error.

For examples of a class with no constructor, this:
```
class Player  
end
```
would be identical to this:
```
class Player  
	constructor()  
	end  
end
```

A constructor is a special instance member whose sole purpose is initialization. Unlike a factory function, it is automatically invoked when a class object is called and cannot return a value.

#### Access Control
Fields are public only when explicitly marked `public`.  Private members are accessible only from within the lexical body of the enclosing class declaration.
```luau
class Player
    private coins: number = 0

    function Coins()
        return self.coins
    end
end

local p = Player("Builderman")
print(p.coins) -- error
```
Likewise, methods cannot be defined outside the class declaration. This allows encapsulation to be enforced lexically, guarantees a stable object layout, and simplifies compiler optimization.
```luau
function Player:GetCoins() -- syntax error
	return self.coins
end
```

#### Class Objects

The action of evaluating a class definition statement introduces a *class object* in the module scope.  A class object is a value that serves as a factory for instances of the class and as a namespace for methods that are defined on the class.

Class objects behave like class instances in most ways, but are always `const` and frozen.

Taking references to class methods via `ClassName.method` syntax is allowed so that classes can easily compose with existing popular APIs:

```luau
local n = pcall(SomeClass.getName, someClassInstance)
```

To construct an instance of a class, call the class object as though it were a function. If the class defines a constructor, the provided arguments are forwarded directly to that constructor. If no constructor is defined, a default zero-argument constructor is synthesized. Constructors initialize the newly allocated instance and do not return values.

The top type of all class objects is named `class`.  `type()` and `typeof()` return `"class"` when passed a class object.

#### Class Instances

Class instances are a new type of value in the VM.  They are similar but not quite the same as tables.  They have no array part, for instance.

`pairs`, `ipairs` , `getmetatable`, and `setmetatable` do not work on class instances.  They also cannot be iterated over with the generic `for` loop. (unless the class implements `__iter`)

Reading or writing a nonexistent class property raises an exception.  This makes it easy to disambiguate between a nonexistent property and a property whose value is nil.

We introduce a new top type for class instances: `object`.  The builtin `type()` and `typeof()` functions return `"object"` for any class instance.  We chose this over having them return the class name because class names do not have to be globally unique (they must only unique within a single module) and because we do not want to make it possible for classes to impersonate other types.

Private fields participate in object layout normally but are inaccessible outside the lexical class declaration.

```luau
class Cls end
local inst = Cls {}

type(Cls) == "class"
typeof(Cls) == "class"

type(inst) == "object"
typeof(inst) == "object"
```

Comparisons between object instances is the same as with tables: If `__eq` is not defined, object comparisons use physical (pointer) equality.  `__eq` is only invoked if both operands are the same type.

#### The `class` library

We introduce a new global library `class`.  Its contents are

```luau
local class: {
    isinstance: (o: object, C: class) -> boolean,
    classof: (o: object) -> class?,
}
```

This library also serves as an obvious extension point for future features like reflection.

The function `class.isinstance(o, Class)` returns `true` if the object `o` is an instance of `Class`.  At runtime, it raises an exception if the second argument is not a class object.  If the first argument is not a class instance, `class.isinstance` returns false.  (Ex: `class.isinstance(5, MyClass)`)

### Type System

Class definitions also introduce a new type to the type environment.

Unlike tables, which are structurally typed, class types are nominal.  Two different classes with identical fields are treated as distinct types.

Inferring the types of class fields is fraught with difficulty, so un-annotated fields are given the type `any`.

The type introduced by a class definition is available anywhere in the source file. Within instance methods and constructors, the implicit `self` binding is typed as the enclosing class.

The `class.isinstance` function participates in refinement:

```luau
function foo(p: unknown)
    if class.isinstance(p, Point) then
        return {p.x, p.y} -- no error here
    end
end
```

Each class object is a singleton instance of an unnamed type.  If needed, it is easy to access via `typeof(TheClass)`.  Class object types are all subtypes of the top `class` type.

### Semantics

Class definitions are Luau statements just like function definitions.

The action of a class definition statement is to allocate the class object, define its functions and properties, and freeze it.  Consequently, a class cannot be instantiated before this statement is executed.

Unlike the accepted proposal, class declarations are not runtime-hoisted. They behave like ordinary Luau declarations. Forward type references continue to work through the existing type solver.

Static analysis also considers the class's type to be global to the whole module so that it can appear in any type annotation anywhere in the script.

An example:

```luau
-- illegal: MyClass is not yet defined
local a = MyClass()

-- OK: MyClass can appear in any type annotation anywhere
function use(c: MyClass)
end

function create()
    -- OK as long as this function is invoked after the class definition statement
    return MyClass()
end

-- We can't statically catch this in the general case, but this will fail at runtime!
create()

class MyClass
end

local b = MyClass() -- OK
local c = create() -- OK
```

Because class definition is a statement, class methods can capture upvalues just like ordinary functions do.

```luau
local globalCount = 0

class Counter
    public count: number

    constructor()
        self.count = globalCount
        globalCount += 1
    end
end
```

When a class object is invoked, the runtime order is as follows:

1. Allocate a new instance of the class.
2. Initialize fields using any default initializers.
3. Invoke the class constructor, if one exists.
4. Return the initialized instance.

### Out of scope for now

#### Const/Static fields

The keyword `static` will be reserved for future RFCs. 

#### Generic classes

This is another feature complex enough to demand its own RFC.

#### Inheritance

Inheritance's worth as a programming technique is controversial: the [Fragile Base Class Problem](https://en.wikipedia.org/wiki/Fragile_base_class) can cause significant harm to a project.

Method inlining is also greatly complicated by inheritance.

#### Composition

Whether Luau should support composition-specific language features is also outside the scope of this RFC.

## Compatibility

This proposal intentionally changes several aspects of the accepted Classes RFC:

- dedicated constructors replace table-based construction
- class declarations are no longer runtime-hoisted
- encapsulation is introduced
- instance methods receive an implicit `self`
- classes become closed declarations

No implementation of the accepted RFC currently exists, so these changes do not affect existing Luau programs.

## Drawbacks

This is a really big feature that has lots of moving parts!

We need to introduce multiple new contextual keywords: `class`,`public`, `constructor` and `private`.  We also introduce two new top types `object` and `class`.

Allowing code to grab unbound method references (ie `local m = o.someMethod`) seems risky because it opens the doorway to a lot of difficult-to-optimize dynamism, but it also makes a bunch of nice things like `pcall` work exactly the way developers expect.  We're making the bet here that this does not materially affect our ability to optimize more mundane attribute access or method calls.

The word `class` is doing triple duty under this RFC: It is a contextual keyword, the name of a top-level library, and the name of the top type for class objects.

Object oriented codebases tend to have far more cyclic dependencies between modules because every piece of data is also coupled to a whole bunch of functions that operate on that data.  We are probably going to have to work out a way to relax the restrictions on cyclic module imports.

## Alternatives

### Accepted Class RFC
The currently merged class RFC is much smaller in scope but still lays out the foundation for native OOP in Luau and can still be amended with later RFCs for the features it's currently missing.

### Explicit Self
An explicit `self` can make instance methods more readable but creates boilerplate code. Type annotations for `self` also wouldn't be allowed.

### Constructor syntax
This RFC uses a dedicated `constructor` declaration instead of requiring constructors to be expressed as ordinary methods or metamethod-like hooks.  
  
A dedicated constructor has a few advantages:  
  
- It separates object initialization from normal instance behavior.  
- It avoids making construction look like an ordinary method call.  
- It avoids reserving conventional method names such as `new`.  
- It does not resemble a Luau metamethod, unlike names beginning with `__`.  
- It allows class invocation syntax, `Point(...)`, to have one clear meaning.

Alternatives such as `new` or `__init` would make construction appear to be either a normal method or a metamethod-like hook, even though construction has special language semantics.

## Open Questions
- Should explicit `public` remain required, or should public be the default?
- Should constructorless classes synthesize a default constructor or support table initialization?
- Should `static` be reserved for the future? Are static fields and methods necessary?
- Should `super` and `protected` be reserved despite inheritance's drawbacks?
- Should definite assignment analysis be required, or should initialization rules be simplified?
