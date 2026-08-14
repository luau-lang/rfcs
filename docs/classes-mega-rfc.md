# Classes Mega RFC

## Classes

### Summary

```luau
class Point
    public x: number
    public y

    function length(self)
        return math.sqrt(self.x * self.x + self.y * self.y)
    end

    function __add(self, other: Point)
        return Point.new(self.x + other.x, self.y + other.y)
    end

    function __tostring(self)
        return `Point \{ x = {self.x}, y = {self.y} \}`
    end
end

local p = Point.new { x = 3, y = 4 }
print(`Check out my cool point: {p}  length = {p:length()}`)
```
### Motivation

* People write object-oriented code.  We should afford it in a polished way.
* Accurate type inference of `setmetatable` has proven to be very difficult to get right.  Because of this, the quality of our autocomplete isn't what it could be.

### Design

Class definitions are a block construct.  They can only be written at the topmost scope.  `export class X` is allowed.

Defining two classes with the same name in the same module is forbidden.

Within a class block, two declarations are allowed: Fields and methods.

For now, fields are introduced with the new `public` keyword.  Private fields will be described in their own section of this document when they are ready.

Class fields are not strictly required to have an annotation, but it is very strongly encouraged.  Unannotated fields will have type `any` and strict mode will flag the field with a warning.

Methods are introduced with the familiar `function` keyword.  `public function f()` is also permitted.

Methods defined on class objects can be accessed either via `Class.method()` or `instance:method()` syntax.

If a method's first argument is named `self`, it should be invoked with the familiar `instance:method()` call syntax.  This is not strictly required, but the compiler and optimizers may deoptimize code that doesn't.  Type annotations on the `self` parameter are not allowed.

If a method accepts no arguments or if its first argument is not named `self`, it should be invoked via `Class.method()` syntax.  This is the same as "static methods" from other languages.

To create a new instance of a class, invoke its static method `.new`. It accepts one argument: A table that describes the initial values of all its properties. Classes are forbidden from expressly defining a `.new` method.  It is reserved by the language.

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

For forward-compatibility, it is a syntax error to define any other method whose name starts with two underscores.

### Class Objects

The action of evaluating a class definition statement introduces a *class object* in the module scope.  A class object is a value that serves as a factory for instances of the class and as a namespace for any functions that are defined on the class.

Class objects behave like class instances in most ways, but are always `const` and frozen.

Taking references to class methods via `ClassName.method` syntax is allowed so that classes can easily compose with existing popular APIs:

```luau
local n = pcall(SomeClass.getName, someClassInstance)
```

To construct an instance of a class, call the class object's `.new` static method.  It accepts a single argument: a table-like value that contains initial values for all the fields.  While it will typically be most useful to pass a table literal to this function, that isn't the only use.  For example, any class object can be shallowly cloned by passing it to its class constructor: `local clone = MyClass.new(original)`

The top type of all class objects is named `class`.  `type()` and `typeof()` return `"class"` when passed a class object.

### Class Instances

Class instances are a new type of value in the VM.  They are similar but not quite the same as tables.  They have no array part, for instance.

`pairs`, `ipairs`, `getmetatable`, and `setmetatable` do not work on class instances.  They also cannot be iterated over with the generic `for` loop. (unless the class implements `__iter`)

Reading or writing a nonexistent class property raises an exception.  In contrast to tables, this makes it easy to disambiguate between a nonexistent property and a property whose value is nil.

We introduce a new top type for class instances: `object`.  The builtin `type()` and `typeof()` functions return `"object"` for any class instance.  We chose this over having them return the class name because class names do not have to be globally unique (they must only unique within a single module) and because we do not want to make it possible for classes to impersonate other types.

```luau
class Cls end
local inst = Cls.new {}

type(Cls) == "class"
typeof(Cls) == "class"

type(inst) == "object"
typeof(inst) == "object"
```

Comparisons between object instances is the same as with tables: If `__eq` is not defined, object comparisons use physical (pointer) equality.  As with Lua, `__eq` is only invoked if both operands share the same definition.

### The `class` library

We introduce a new global library `class`.  Its contents are

```luau
local class: {
    isa: (o: object, C: class) -> boolean,
    classof: (o: object) -> class?,
}
```

This library also serves as an obvious extension point for future features like reflection.

The function `class.isa(o, Class)` returns `true` if the object `o` is an instance of `Class`.  At runtime, it raises an exception if the second argument is not a class object.  If the first argument is not a class instance, `class.isa` returns false.  (eg `class.isa(5, MyClass)`)

### Type System

Class definitions also introduce a new type to the type environment.

Unlike tables, which are structurally typed, class types are nominal.  Two different classes with identical fields are treated as distinct types.

Inferring the types of class fields is fraught with difficulty, so un-annotated fields are given the type `any`.

The type introduced by a class definition is available anywhere in the source file.

The `class.isa` function participates in refinement:

```luau
function foo(p: unknown)
    if class.isa(p, Point) then
        return { p.x, p.y } -- no error here
    end
end
```

Each class object is a singleton instance of an unnamed type.  If needed in a type context, it is easy to access via `typeof(TheClass)`.  Class object types are all subtypes of the top `class` type.

### Semantics
Class definitions are Luau statements just like function definitions.

The action of a class definition statement is to allocate the class object, define its functions and properties, and freeze it.  Consequently, a class cannot be instantiated before this statement is executed.

Because class definition is a statement, class methods can capture upvalues just like ordinary functions do.

```luau
local globalCount = 0

class Counter
    public count: number

    function create()
        local count = globalCount
        globalCount += 1
        return Counter.new { count = count }
    end
end
```
### Hoisting

We do, however, *hoist* the class identifier's binding to the top of the script so that it can be referred to within functions or classes that lexically appear before the class definition.  This makes it easy and straightforward for developers to write classes or functions that mutually refer to one another.

Static analysis also considers the class's type to be global to the whole module so that it can appear in any type annotation anywhere in the script.

An example:

```luau
-- illegal: MyClass is not yet defined
local a = MyClass.new {}

-- OK: MyClass can appear in a type annotation anywhere
function use(c: MyClass)
end

function create()
    -- OK as long as this function is invoked after the class definition statement
    return MyClass.new {}
end

-- We can't statically catch this in the general case, but this will fail at runtime!
create()

class MyClass
end

local b = MyClass.new {} -- OK
local c = create() -- OK
```

### Drawbacks

This is a really big feature that has lots of moving parts!

We need to introduce multiple new contextual keywords: `class` and `public` to start and `private` later.  We also introduce two new top types `object` and `class`.

Allowing code to grab unbound method references (ie `local m = o.someMethod`) seems risky because it opens the doorway to a lot of difficult-to-optimize dynamism, but it also makes a bunch of nice things like `pcall` work exactly the way developers expect.  We're making the bet here that this does not materially affect our ability to optimize more mundane attribute access or method calls.

The word `class` is doing triple duty under this RFC: It is a contextual keyword, the name of a top-level library, and the name of the top type for class objects.

Object oriented codebases tend to have far more cyclic dependencies between modules because every piece of data is also coupled to a whole bunch of functions that operate on that data.  We'll also build out better support for cyclically-dependent modules.  See https://github.com/luau-lang/rfcs/blob/master/docs/support-for-cyclic-requires.md

## Inheritance

### Summary

```luau
open class Animal
    public species: string

    function __tostring(self)
        return "I am an animal."
    end

    function live(self)
        return "I am alive"
    end
end

class Cat extends Animal
    public meowMult: number

    function __tostring(self): string
        return `{Animal.__tostring(self)} I am a cat.`
    end

    function meow(self): string
        return string.rep("meow\n", self.meowMult)
    end
end

local dog = Animal.new { species = "Canis familiaris" }
local cat = Cat.new { species = "Felis catus", meowMult = 3 }
```

### Motivation

Other languages with support for object-oriented programming support inheritance in some way. We (the Luau team) have even written libraries which would benefit from having classes with nominal inheritance. Inheritance also makes it easier to write more modular code and provides another guidepost for programmers coming from other languages. Additionally, most Lua and Luau libraries which provide object oriented programming affordances make implementation inheritance available in some way.

### Syntax

We introduce two new contextual keywords: `open` and `extends`.

We observe that, in real programs, almost all classes are either not intended to participate in inheritance at all, or they are expressly crafted to be used as a base class.  We therefore require developers to use the `open class` keyword-pair to declare a class that can be inherited from.

Attempting to inherit from a class that is not `open` is an error.  Type checking will flag it and the runtime will raise an exception.

To extend an open class, use the `extends` keyword.

If a class has a base class and is also itself intended to be used as a base class, the developer must write both: `open class Second extends First`

If a method is defined in a superclass but not reimplemented by a subclass, then the subclass inherits its implementation.

Any fields declared in a superclass are present in its subclasses. As a result, instantiating a subclass must specify the union of fields in the subclass and its ancestry chain, even if they are not explicitly mentioned in the subclass’s declaration. For example:

```luau
open class Point
    public x: number
    public y: number
end

class ThreeDPoint extends Point
    public z: number
end

local threedpoint = ThreeDPoint.new { x = 0, y = 0, z = 0 }
local erroneous = ThreeDPoint.new { z = 1 } -- Type error.  x and y are uninitialized.  They will be nil at runtime.
```

Subclasses are forbidden from redeclaring fields declared in their superclasses. Such a redeclared field would need to type invariantly against the superclass field to maintain soundness anyway. Additionally, this reduces ambiguity for programmers coming from other languages, such as C++, where shadowed fields exist independently (i.e. `A::field` vs. `B::field`, where `B` subclasses `A`). In the case that private fields are added to classes, we expect this restriction to apply only to public fields.

A subclass can redefine a method present in its superclass. However, the method declared in the subclass must be a subtype of the method in the superclass.  That is, a child class can override a parent class method with a function that is more permissive than that of the base class method, but not less.

```luau
open class Base
    function method(self, x: number|string): {}?
        print(x)
        return {}
    end
end

class ChildOne extends Base
    -- OK: This can handle anything Base.method can handle and only returns
    -- things that callers to Base.method are expecting.
    function method(self, x: number|string?): {x: string}
        print(x)
        return {x="hello"}
    end
end

class ChildTwo extends Base
    -- Error: Base.method can be called with strings.  Additionally,
    -- callers to Base.method are not prepared to handle a string result.
    function method(self, x: number): {} | string
        print(x)
        return ""
    end
end
```

### Name resolution

Methods (including metamethods) defined on base classes are accessible via the derived class, with the exception of `__init`, which has its own set of rules described in the Constructors section.

```luau
open class Base
    function foo() end
end

class Child extends Base
end

Child.foo() -- ok
```

Class instances are the same:

```luau
local c = Child.new({})
c.foo() // OK
```

Above, we mentioned that `__eq` between two class instances is only invoked if both operands share it. This rule is still followed in the presence of inheritance:

```luau
open class Point
    x: number
    y: number

    function __eq(self, other)
        return self.x == other.x and self.y == other.y
    end
end

class NamedPoint extends Point
    name: string
end

local p = Point.new(0, 0)
local np = NamedPoint.new(0, 0, "Bob")

print(p == np) -- invokes Point.__eq and evaluates to true, not a proof
```

### Declaration Order

The Classes section above states that class names are hoisted and so can be used before the class declaration has been evaluated.  This is unchanged by this section, but carries with it a consequence that is important to call out:

Class declarations have effects at the top level.  Therefore, a class cannot inherit from another class that occurs lexically after it within the module.

```luau
class Child extends Base -- error: Base is nil here!
end

open class Base
end

class SecondChild extends Base -- OK
end
```

Developers are advised to order their class declarations accordingly.  We will likely need a lint rule to detect this case.

### Library Changes

The `class` library's `isinstance` member will support subclasses.

```luau
class.isinstance(cat, Cat) -- true
class.isinstance(cat, Animal) -- true
```

Notably, refinements triggered by `class.isinstance` say a little bit more with inheritance into the mix: When the refinement is negated, we will sometimes infer some types that are a bit more interesting than expected:

```luau
function foo(a: Animal)
    if class.isinstance(a, Cat)
        -- a : Cat
    else
        -- a : Animal & ~Cat
    end
end
```

The `class` library's `classof` member will return the most specific class object that corresponds to the passed object's dynamic type:

```luau
function get_class(a: Animal)
    return class.classof(a)
end

get_class(Cat.new {...}) -- returns Cat
```

### Type System

For any subclass which inherits, the type of its instances subtypes nominally against the type of its superclass's instances. For example, the following typechecks:

```luau
function printSpecies(animal: Animal)
    print(animal.species)
end

printSpecies(cat)
```

However, class objects never have any subtyping relationship.  For example, the following will not typecheck:

```luau
function initialize(cls: typeof(Animal))
    return cls.new { species = "Homo sapiens" }
end

initialize(Cat) -- type error
```

The reason for this is simple: Subtyping of the constructor is not at all guaranteed.

However, constructors are ordinary functions and so can be passed by value if desired.  They obey ordinary function subtyping rules:

```luau
function initialize(factory: () -> Animal)
     return factory()
end

local a = if some_flag
    then initialize(Cat.new)
    else initialize(Elephant.new)
```

#### Typechecking overridden methods

We now revisit a modified version of the initial example of inheritance, this time with annotations on `self` describing what its type might theoretically be (annotations on the `self` parameter are actually syntax errors).

```luau
open class Animal
    function greet(self: Animal, name: string)
        return `Hi {name}, I am an animal.`
    end
end

class Cat extends Animal
    function greet(self: Cat, name: string?): string
        -- Cats always ignore me when I greet them. :(
        return '...'
    end
end
```

We said previously that methods that are redefined in subclasses must subtype the superclass's version. The subtype test for `greet` (with the annotations we've added) goes as follows:

- `(self: Cat, name: string?) -> string <: (self: Animal, name: string) -> string`
- `(self: Animal, name: string) <: (self: Cat, name: string?)` (method arguments are subtyped contravariantly)
- `(self: Animal) <: (self: Cat)` (we subtype corresponding members of type packs)
- `Animal </: Cat` Class types are always subtyped nominally. We define that superclasses do not subtype their subclasses, so this step fails.

To allow subclasses to override methods and have them typecheck, we add a special case to the type checker for overridden methods. More specifically, if we are subtyping two class methods where the first argument is named `self` for both, we skip subtying the first argument. Then, the subtyping test for `greet` proceeds as follows:

- `(self<Cat>, name: string?) -> string <: (self<Animal>, name: string) -> string` (we slightly abuse notation to use `<>` to annotate which version of `greet` is which)
- `(name: string) <: (name: string?)` (we skip `self` and subtype the other method arguments contravariantly as normal)
- `string <: string?` (this holds because the set of values represented by `string` is contained in the set of values represented by `string`)

We acknowledge the lack of a `self` type, which is necessary for methods that need to refer to the current class type even for subclasses. (eg to write a `function clone(self): self ... end`)  This can be proposed in another RFC.

### Drawbacks

We introduce even more keywords: `open` and `extends`.

Adding implementation inheritance reduces optimization opportunities involving class method inlining and dispatch. For example, consider the following:

```luau
function callToString(a: Animal)
    return a.__tostring()
end
```

Since `Animal` is `open`, we can no longer statically determine which method to dispatch within the body of `callToString`. This drawback can be navigated by implementing more advanced optimization techniques such as speculative method inlining. 

### Alternatives

We consider only single implementation inheritance to avoid the diamond problem. This is the same approach that Java takes.

We may consider multiple interface inheritance in the future.

Instead of offering an `open` keyword to allow inheritance, we could follow in Java's footsteps by offering `final` instead.  Our assessment is that opt-in is the correct default.

We considered a `super` keyword, but found the justification for it to be very weak.  The developer can just write the actual name of the base class.  Additionally, we would have to answer some very awkward questions about what happens when the script defines another symbol named `super`. (which seems like a pretty good variable name!)

## Constructors

### Summary

Add user-definable constructors to Luau classes.

### Motivation

Above, we specified that all classes can be constructed by invoking `.new` on the class object with a mapping from fields to values as its sole argument.

This is great for "plain old data" classes and we could consider allowing users to write their own static `.new()` method if they have more exotic construction requirements, but this falls apart in the face of inheritance because a `.new()` factory necessarily couples the actual class instance construction with the field initialization.

Concretely:

```luau
class BasePoint
    public x: number
    public y: number

    function new(): BasePoint
        return BasePoint { x = 0, y = 0 }
    end
end

class DerivedPoint extends BasePoint
    public name: string

    function new(): DerivedPoint
        -- We're stuck!  We cannot implement this function in terms of BasePoint.new()
    end
end
```

Implementation inheritance requires some mechanism by which a class constructor can do the work only to initialize its part of the object for some class that is at some indeterminate point in an inheritance hierarchy.

This is a very well-known problem.  The well-known solution is constructors.

### Syntax

We draw inspiration from Python and allow classes to replace the builtin constructor by defining an `__init` method:

```luau
class A extends B
    public x: number

    function __init(self, x, y)
        B.__init(self, x)
        self.x = x * y
    end
end
```

If a class defines an `__init` function, it is understood to be a constructor.  Classes with user-defined constructors follow different rules.  We define class construction as follows:

Classes are still constructed using the static method `.new`.

Let `T` be a class object that defines a constructor.  When `T.new(...args)` is invoked with any arguments, the following happens:

1. A fresh, uninitialized instance of `T` is allocated.  We'll call it `t` here.  All of its fields are initially `nil` irrespective of any type annotations.  
2. `T.__init(t, ...args)` is invoked.  
3. `t` is produced as the result of the expression.

### The Default Constructor

If a class does not explicitly define a constructor, it is given a default constructor.  The default constructor takes a mapping from property name to value and initializes the newly-created class instance with those properties.  If no argument or `nil` are passed, the default constructor will raise an error.

In strict mode, it is a warning to leave fields uninitialized.

A class must define an explicit constructor if its base class defines one.  Attempting to construct such a class will result in a runtime exception.  Type inference will also warn in this case.

If a class defines no constructor and inherits from another that also defines no constructor, the default constructor will initialize all fields of the class.  For example:

```luau
open class BasePoint
    public x: number
    public y: number
end

class Point3D extends BasePoint
    public z: number
end

local p2 = Point.new { x=3, y=4 }
local p3 = Point3D.new { x=1, y=2, z=3 }
```

The default constructor is a real function just like any other and so it can be explicitly invoked if desired.

```luau
class Point
    public x: number
    public y: number

    function reset(self)
        self:__init {x=0, y=0}
    end
end
```

### Typechecking

In order to make uninitialized data unobservable, constructors are required to follow some strict typing rules:

* The first argument of a constructor must be called `self`.  
* A class must define a constructor if its base class defines one.
* If a class has a base class, the class constructor must invoke its base class's constructor before reading or writing to `self` or any of its properties.
* A constructor must unconditionally initialize all of its fields before it can pass `self` to any function.  
    * The delegating call to the base class constructor is of course exempt from this.  `BaseClass.__init(self, args)`  
    * This restriction also includes all method calls (eg `self:something()`)  
    * As a special exception, fields whose types are supertypes of `nil` are exempt from this requirement and are always considered to have been implicitly initialized with `nil`.  Explicit initialization of such fields is of course permitted.  
    * Additionally, any subexpression where `self` is explicitly cast is exempt from this. (eg `foo(self :: any)` or even `foo(self :: FooType)`)  We offer this as an intentional way for a developer to override the type system if they need to.  
* A constructor must not refer to any field before it has been initialized.

Once the base class has been called and all fields are initialized, constructors can do anything.

These rules are all enforced only by type checking.  When the program is run, uninitialized fields will have the value `nil` no matter what their types might say. This is consistent with what happens when a class `T` does not define `__init`, and `T.new(t)` is invoked where `t` does not have a key-value pair for every one of `T`'s fields.

### Drawbacks

Constructors add more complexity to the language.  Some classes can be initialized via `T.new { x = x, y = y}` syntax and others must be initialized with different kinds of arguments.

We intentionally only check that fields are initialized in type checking.  This means that our runtime still has to be able to cope with fields that have been left uninitialized.

Some developers will feel inconvenienced by the restrictions on uninitialized class fields.  The current rules do not, for instance, permit a developer to write a helper function that partially (or completely) initializes a new class instance.

### Alternatives

We could follow in Python's footsteps and do away with the default `T.new {}` constructor, but this means that developers have to write a lot of dull code in the typical "plain old data" case:

```luau
class Point
    public x: number
    public y: number

    -- This whole function is pointless ceremony
    function __init(self, x, y)
        self.x = x
        self.y = y
    end
end
```

Python does away with this in some cases via the `@dataclass` decorator.

We could make field initialization more logical by adding C++-style initializer syntax.  We're already adding a lot of syntax and we don't think it's worth it for us.

The choice to construct instances via a `.new` static function is motivated by Lua libraries that effect objects.  We sacrifice the ability for classes to define their own `.new` method but in exchange, code using classes looks more consistent with what came before.

# Appendix

This appendix contains considerations that are related more to the underlying implementation of each feature, rather than the features themselves.

## Classes

### Performance-related Motivation
These are performance-related bonuses that fall out of implementing classes.

* A construct with a fixed shape and a completely locked-down metatable will open up optimization opportunities that could improve performance:
    * If classes can only be declared at the top scope, then we know that each method of each class has exactly one instance.  This makes it simple for the compiler to know the exact function that will be invoked for any method call expression.
    * If a value is known to be an instance of a particular class, the bytecode compiler should be able optimize method calls to skip the whole `__index` metamethod process and instead generate code to directly call the correct method.
    * By the same token, method calls can be inlined more aggressively.  Particularly self-method calls eg `self:SomeOtherMethod()`
    * Field accesses can compile to a simple integral table offset so that the VM doesn't need to do a hashtable lookup as the program runs.
    * Since every instance of a class has the same set of properties, we can split the hash table: The set of fields can be associated with the class and instances only need to carry the values of those fields.  We think this can improve performance by improving cache locality.

## Inheritance

### Memory Layout

Classes have a fixed layout.  In the VM, the mapping from name to offset is stored with the class object, not with individual class instances.

In the presence of inheritance, we specify one additional rule: All inherited properties of base classes exist in memory at the same offsets.  Properties of the derived class occur afterward.

This means that, if the VM knows that some object is a subtype of some class type, it can access a particular known field at a particular known offset.  That offset will be correct no matter what the concrete type is.

## Constructors

### Typechecking

For v1, type inference does not attempt to track fields that are initialized within conditional constructs like `if` statements or loops.  Fields must be initialized directly at the function scope. 