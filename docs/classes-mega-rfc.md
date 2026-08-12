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

For now, fields are introduced with the new `public` keyword.  We also plan to eventually offer private fields, but it is sufficiently complex that it merits its own RFC.

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

To construct an instance of a class, call the class object's `.new` static mehtod.  It accepts a single argument: a table-like value that contains initial values for all the fields.  While it will typically be most useful to pass a table literal to this function, that isn't the only use.  For example, any class object can be shallowly cloned by passing it to its class constructor: `local clone = MyClass.new(original)`

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

Comparisons between object instances is the same as with tables: If `__eq` is not defined, object comparisons use physical (pointer) equality.  `__eq` is only invoked if both operands are the same type.

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

Object oriented codebases tend to have far more cyclic dependencies between modules because every piece of data is also coupled to a whole bunch of functions that operate on that data.  We are probably going to have to work out a way to relax the restrictions on cyclic module imports.

## Inheritance

## Constructors

## Private Fields

# Appendix

## Performance-related Motivation
These are performance-related bonuses that fall out of implementing classes.

* A construct with a fixed shape and a completely locked-down metatable will open up optimization opportunities that could improve performance:
    * If classes can only be declared at the top scope, then we know that each method of each class has exactly one instance.  This makes it simple for the compiler to know the exact function that will be invoked for any method call expression.
    * If a value is known to be an instance of a particular class, the bytecode compiler should be able optimize method calls to skip the whole `__index` metamethod process and instead generate code to directly call the correct method.
    * By the same token, method calls can be inlined more aggressively.  Particularly self-method calls eg `self:SomeOtherMethod()`
    * Field accesses can compile to a simple integral table offset so that the VM doesn't need to do a hashtable lookup as the program runs.
    * Since every instance of a class has the same set of properties, we can split the hash table: The set of fields can be associated with the class and instances only need to carry the values of those fields.  We think this can improve performance by improving cache locality.