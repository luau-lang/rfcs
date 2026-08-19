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

Public fields are introduced with the new `public` keyword.  Private fields are described in their own section of this document.

Class fields are not strictly required to have an annotation, but it is very strongly encouraged.  Unannotated fields will have type `any` and strict mode will flag the field with a warning.

Methods are introduced with the familiar `function` keyword.  `public function f()` is also permitted. Both syntaxes declare public methods.

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

Subclasses are forbidden from redeclaring fields declared in their superclasses. Such a redeclared field would need to type invariantly against the superclass field to maintain soundness anyway. Additionally, this reduces ambiguity for programmers coming from other languages, such as C++, where shadowed fields exist independently (i.e. `A::field` vs. `B::field`, where `B` subclasses `A`). This restriction applies only to public fields.

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

## Private Class Fields

### Summary

We add private fields and methods.

Throughout this PR, we use “properties” to mean the set of a class’s field and methods.

```
class Point
    private _x: number
    private _y: number

    public function getX(self)
        return self._x
    end

    public function getY(self)
        return self._y
    end

    private function _computeMagnitude(self)
        return math.sqrt(self._x ^ 2 + self._y ^ 2)
    end

    public function getMagnitude(self)
        return self:_computeMagnitude()
    end

    public function __init(self, x, y)
        self._x = x
        self._y = y
    end
end
```

Private properties are declared with the `private` keyword and must be prefixed with an underscore `_`. We recognize that this is a departure from how other languages use just a `private` keyword. We have a strong reasoning for this divergence, and make the case for it in the Alternatives section.

Private properties can only be accessed by the methods of the class in which they are defined. 

Private properties are not visible outside the class in which they are declared. As a result, instances of classes with private fields can only be directly constructed within the class where they are declared. If such a class is to be instantiated elsewhere, it must define a constructor.

Subclasses do not have access to their parent’s private properties. This means that private properties can be shadowed. A base class and a derived class can both have private properties `_x` that do not collide or interact in any way. A derived class can also define a public property `x` that "shadows" a base class's `_x` field.

### Motivation

An important feature for library authors is a way to hide implementation details and force users through a curated interface that is intentionally kept stable.

There are two ways to afford this in Luau today, and both of them have problems:

#### Closures

First, data can be hidden away inside a closure.  This is effective, but forces library authors into a particular coding style that isn't the most efficient: Every instance of every method is a distinct closure.

```
function Counter()
    local private_count = 0

    return {
        method = function()
            local result = private_count
            private_count += 1
            return result
        end
    }
end
```

#### Convention

Secondly, library authors can use a naming convention (like `_prop`) to indicate data that is not part of the public interface and ask developers not to use those symbols.  This is more efficient because it allows method implementations to be shared between instances and so introduces way fewer closures, but tends not to work very well because users inevitably write code that depends on this state.

Roblox itself has used this technique and found itself forced to permanently preserve supposedly "private" properties.

### Syntax

Methods and fields can both be declared private by using the new `private` keyword. Private property names must be prefixed with “`_`”. A private field can only be accessed by methods of that class. Private methods can only be called by methods of that class. Static methods can also be private.

As is the case with public properties, using keywords as the names of private properties is forbidden.

To avoid complications involved with the notion of a private metamethod or metatable and parsing ambiguity, declaring private property that begins with `__` is forbidden. Defining a private property named `_new` is also disallowed to maintain forwards compatibility with the case that we introduce private constructors.

Public fields will continue to be indicated with the `public` keyword. Methods with no access modifiers will continue to be interpreted as `public`. With the exception of metamethods, public properties may not begin with `_`. We considered having no keyword at all for non-private fields, but the lack of required separators between fields makes it difficult to introduce contextual keywords in the future, such as for `const` or `protected` fields.

To access private fields and methods, we introduce two new operators, `._` and `:_`, which are only parsed within method bodies. The only way that private properties can be accessed is using these operators: `obj._privateField` or `obj:_privateMethod()`.

We could theoretically elide the requirement that property names begin with `_`, but we include it so that private property accesses look like normal accesses on properties that happen to begin with `_`, and to align conventions that already exist in many Luau libraries. 

This syntax also makes it easier to disambiguate between public and private properties, especially within large classes where a property access might be lexically far from where the property was declared.

The `._` and `:_` operators collide with the syntax for indexing into a table with a key that begins with `_`. Within the body of a method, indexing notation (`t[“_propName”]`) must be used instead to access the property of a table prefixed with `_`. Outside method bodies, indexing for properties prefixed with `_` is available using the `.` and `:` operators as normal. 

While this restriction imposes some inconvenience, it does not restrict the semantic space of expressible programs in any way.

A class’s private property names cannot shadow any of its or any ancestor’s public properties.

### Semantics

Inside a method body, the private fields of any instance of the enclosing class (not just self) are available. This means in the following, the `__eq` metamethod can access `other`’s private properties. (Recall that the `__eq` metamethod is only invoked for two values of the same type).

```
open class Point
    private _x: number
    private _y: number

    ...

    -- Note that no annotation is required for 'other' here
    function __eq(self: Point, other)
        return self._x == other._x and self._y == other._y
    end
end
```

#### Private Fields and Inheritance

It is safe for private class properties to be shadowed, since child classes cannot access private fields and methods of their ancestors.

An instance of a class has values for the private properties of every class that it is a member of. This means that instances of `Point` can still be compared to `ThreeDPoint`.

```
class ThreeDPoint extends Point
    private _z: number
end
```

Type inference can also be extended so that private field access in a method body applies a subtyping constraint between the object being accessed and the enclosing class. In the above example, `other` would be inferred to have supertype `Point`.

The operators `._` and `:_` allow child classes to shadow private properties with public ones:

```
open class Base
    private _x: number

    function compare(self, other)
        return self._x == other._x
    end
end

class Derived extends Base
    public x: number
end

local a = Base.new(...)
local b = Derived.new(...)

-- This unambiguously compares a._x and b._x
local res = a:compare(b)
```

#### Private Fields and Constructors

We have two overarching goals for how constructors should interact with private fields:

- Minimize how often instances can have uninitialized state  
- Don’t make behavior too complex to reason about.

To these ends, classes that have private properties must explicitly define a constructor.  The default constructor will raise a runtime exception when called.  This is necessary because the default constructor would otherwise need to initialize private fields--fields that users of the class shouldn't even be aware of\!

Note that this restriction interacts with the existing requirement that if a base class defines a constructor, then any derived class must also do so. Consequently, if a class inherits from a base class with private fields, then it must now define a constructor.

We considered making the default constructor private if the class has any private fields, but this requires either allowing a single instance where a private method be callable without `._` or adding `._new` to the language. We also considered adding private constructors, declared via `private __init` and invoked via `._new`, but we opted against it, both to align with the second goal above and to limit the scope of this RFC.

### Drawbacks

As mentioned above, table accesses are slightly complicated by the addition of the `._` and `:_` operators to method body contexts.

### Alternatives

#### No Leading Underscore

One alternative we considered is to use a conventional `private` keyword without any requirement that fields start with an underscore.

This is extremely tempting, but our requirement that private fields be able to shadow interacts with the gradual type system in very unfortunate ways.

For instance, in the following code, the reference to `other.x` could either refer to a private field of a `Base` instance, a public field of some other class, or a table key:

```
open class Base
    private x: number

    function compare(self, other)
        return self.x == other.x
    end
end

class Derived extends Base
    public x: number
end

local a = Base.new(...)
local b = Derived.new(...)

-- Should this compare Base.x to Base.x, or Base.x to Derived.x?
local res = a:compare(b)
local res2 = a:compare({x=5})
```

In a prior revision, we considered having the VM do a runtime test to figure it out on the fly, but this hides a very scary maintenance bug:

```
open class BaseClass
    private x: number

    function __init(self, x)
        self.x = x
    end

    function equalTo(self, other)
        return self.x == other.x
    end
end

class ThirdClass -- extends BaseClass
    public x: number

    function test(self, other: BaseClass)
        if other:equalTo(self) then
            print('equal')
        else
            print('inequal')
        end
    end
end

local b = BaseClass.new(4)
local third = ThirdClass.new { x=2 }

third:test(b)
```

As-written, this code would be completely fine:  When `third:test(b)` is invoked, the `Base.equalTo` method reads the private `x` field of `b` and compares it to the public `x` field of `third`.

However, something spooky happens if the author updates the definition of `ThirdClass` to extend `BaseClass`: The `test` method does something completely different now *because the `equalTo` method now does something completely different*.

This is pretty scary\!

Our stance is that, while it is unfortunate to take `._` as a special operator, it hits an important sweet spot: Developers are already used to prefixing nonpublic fields with underscores and it looks much nicer than a higher-profile glyph like `#`.

Asking authors to write `o["_foo"]` every once and awhile seems like a worthy tradeoff.

#### Secret Property Names

Something we considered was replacing private field labels with secret opaque key values.  For instance:

```
class A
    #x: number

    function __init(self)
        self.#x = 0
    end

    function get_x(self)
        return self.#x
    end
end

-- ... would be rewritten as ...

const x_symbol = {}

class A
    [x_symbol]: number

    function get_x(self)
        return self[x_symbol]
    end
end
```

The reasoning goes that, if you can't iterate over the object, then you can't get at the secret token and therefore you can't read the property.

However, this approach raises an issue: `x_symbol` could now be visible from another module. To mitigate this, each class could maintain a mapping of private property names to key values, with some extra information in the keys to prevent collisions:

```
-- module1.luau
class A
    #x: number
    function __init(self) self.#x = 0 end
end

-- main.luau
local module = require("./module1")

class B
    #x: number
    function __init(self) self.#x = 3 end
end

class A
    #x: number
    function __init(self) self.#x = 1 end
    function printX(self)
        local n = math.random()
        local obj = if n > 0.67 then self elseif n > 0.33 then module.A.new() else B.new()
        print(obj.#x)
    end
end

-- ... would become ...   
     
-- module1.luau
class A [private_props: { module1_A_x = {} }]
    [A.private_props.module1_A_x]: number
    function __init(self) self[A.private_props.module1_A_x] = 0 end
end

-- main.luau
local module = require("./module1")

class B [private_props: { main_B_x = {} }]
    [B.private_props.main_B_x]: number
    function __init(self) self[B.private_props.main_B_x] = 3 end
end

class A [private_props: { main_A_x = {} }]
    [private_props[main_A_x]]: number
    function __init(self) self[A.private_props[main_A_x]] = 1 end
    function printX(self)
        local n = math.random()
        local obj = if n > 0.67 then self elseif n > 0.33 then module.A.new() else B.new()
        print(obj[A.private_props[main_A_x]]) -- errors if obj isn't an instance of main.A
    end
end
```

Property access on a private field `a.#x` where `a` is an instance of `A` would: construct the key for `private_props` by concatenating the chunk name, class name, and prop name (say `init_A_x`), index into `A.private_props` for the opaque key value, and use that key value to finally index into `a`.

This approach would have the advantage that we wouldn’t need to check that `a` is an instance of the correct class at runtime, but introduces added complexity, not to mention the added runtime and memory cost of a mapping to hold the opaque key values. 

#### Property rewriting

Another idea we considered was a rewrite rule for properties whose names follow some convention.  Like `_foo`.  We would extend the language to automatically rewrite each such property with a module-local, anonymous name that is unutterable elsewhere.  For instance:

```sql
export a = { _privateKey = 0 }
export b = a._privateKey
a._privateKey += 1

-- to

local _privateKey = { name="_privateKey" }

export a = { [_privateKey] = 0 }
export b = a[_privateKey]
a[_privateKey] += 1
```

This is quite easy to implement and works orthogonally to classes, but is a significant backwards-compatibility break.

#### Restrict private field access to `self`

We considered allowing private field access on only `self`, but this is quite restrictive.

#### Module-specific `private` fields

One possibility was for `private` fields to be accessible from throughout the module in which they’re defined, but this permissiveness seems more appropriate for something like a `friend` access modifier, and the Luau VM would require significant reworking to make this work.

#### Private constructors

We considered adding private constructors, declared via `private __init` and invoked via `._new`.

If a class defines a private method `__init` or has at least one private property, the class would instead have a private builtin static method `new`, which could be used by other methods of the class.  Concretely:

```
class Point
    private _x: number
    private _y: number

    function zero(): Point
        return Point._new { x = 0, y = 0}
    end
end
```

We've decided not to implement this at this time because such a private constructor can only be called by public methods of the class, so the encapsulation benefit here is pretty minor.  In order to offer more, we would have to introduce some mechanism for a symbol to be private to a larger organizational unit like a package. We are opting not to open this particular can of worms just yet.

#### Protected fields

In the future, we may add `protected` fields that are accessible to classes which inherit from the class in which they are declared. Note that the syntax we have chosen is compatible with `protected` if we also prefix `protected` fields with `_`:

```
class Point
    protected _x: number
    protected _y: number
end

class ThreeDPoint extends Point
    protected _z: number
    
    function __init(self, x, y, z):
        self._x = x
        self._y = y
        self._z = z
    end
end
```

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

## Private Class Fields

Here, we describe a possible implementation strategy.

### `GET/SETCLASSFIELD`

Accessing private fields can be implemented via a pair of new `GET/SETPRIVATEFIELD` (actual name TBD) instructions. They are emitted if and only if the bytecode compiler observes `._` or `:_`, and will have roughly the following format:

| Slot | Content |
| :---- | :---- |
| A | Target/source register |
| B | Register containing class instance |
| C | Register containing expected class |
| AUX\[0:14\] | Private property index We will restrict the number of private instance properties and private static properties a class can have to 32,768 each |
| AUX\[15\] | Whether property is instance property or static property |

We restrict our usage of AUX to just 16 bits to leave room for potential future changes.

At runtime, `GET/SETPRIVATEFIELD` will perform a tag check to ensure that `B` is both a class object and an instance of the correct class. Since `._` and `:_` can only occur inside class method bodies, the bytecode compiler can statically determine which class an instance should belong to. Additionally, hoisting means that the relevant `LuauClass*` will be present in a register, which the compiler can include in `C`.

To facilitate statically computing the index of private properties, we will add a `TValue` array to `LuauObject` that will hold the values of private fields, and another `TValue` array to `LuauClass` to hold the values of private static members (currently only methods). The indices into each array are known at compile time since classes cannot access the private properties of their ancestors.  
