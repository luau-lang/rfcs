# Classes

## Summary

```luau
class Point
    public x: number
    public y

    function length(self)
        return math.sqrt(self.x * self.x + self.y * self.y)
    end

    function __add(self, other: Point)
        return Point { x = self.x + other.x, y = self.y + other.y }
    end

    function __tostring(self)
        return `Point \{ x = {self.x}, y = {self.y} \}`
    end

    function new(x, y)
        return Point { x = x, y = y }
    end
end

local p = Point.new(3, 4)
print(`Check out my cool point: {p}  length = {p:length()}`)
```

## Motivation

* People write object-oriented code.  We should afford it in a polished way.
* Accurate type inference of `setmetatable` has proven to be very difficult to get right.  Because of this, the quality of our autocomplete isn't what it could be.
* A construct with a fixed shape and a completely locked-down metatable will open up optimization opportunities that could improve performance:
    * If classes can only be declared at the top scope, then we know that each method of each class has exactly one instance.  This makes it simple for the compiler to know the exact function that will be invoked for any method call expression.
    * If a value is known to be an instance of a particular class, the bytecode compiler should be able optimize method calls to skip the whole `__index` metamethod process and instead generate code to directly call the correct method.
    * By the same token, method calls can be inlined more aggressively.  Particularly self-method calls eg `self:SomeOtherMethod()`
    * Field accesses can compile to a simple integral table offset so that the VM doesn't need to do a hashtable lookup as the program runs.
    * Since every instance of a class has the same set of properties, we can split the hash table: The set of fields can be associated with the class and instances only need to carry the values of those fields.  We think this can improve performance by improving cache locality.

## Design

### Syntax

Class definitions are a block construct.  They can only be written at the topmost scope.  `export class X` is allowed.

Defining two classes with the same name in the same module is forbidden.

Within a class block, two declarations are allowed: Fields and methods.

Fields are introduced with the new `public` keyword.  We also plan to eventually offer `private`, but is sufficiently complex that it merits its own RFC.

Methods are introduced with the familiar `function` keyword.  `public function f()` is also permitted.

If a method's first argument is named `self`, it can be invoked with the familiar `instance:method()` call syntax.  Type annotations on the `self` parameter are not allowed.

If a method accepts no arguments, or if its first argument is not named `self`, it can be invoked via `Class.method()` syntax.  This is the same as "static methods" from other languages.

To create a new instance of a class, invoke it as if it were a function.  It accepts one argument: A table that describes the initial values of all its properties.  If more customization is desired, static factory functions (frequently named `new()` or `create()`) are an easy, familiar way to accomplish this.

Classes can define familiar Luau metamethods like `__add` and `__sub`.  They will work as one would expect.

The following metaproperties are forbidden.  Any attempt to define them is a syntax error:

* `__index`
* `__newindex`
* `__mode`
* `__metatable`
* `__type`

#### Class Objects

The action of evaluating a class definition statement introduces a *class object* in the module scope.  A class object is a value that serves as a factory for instances of the class and as a namespace for any functions that are defined on the class.

Class objects behave like class instances in most ways, but are always `const` and frozen.

Taking references to class methods via `ClassName.method` syntax is allowed so that classes can easily compose with existing popular APIs:

```luau
local n = pcall(SomeClass.getName, someClassInstance)
```

To construct an instance of a class, call the class object as though it were a function.  It accepts a single argument: a table that contains initial values for all the fields.

The top type of all class objects is named `class`.  `type()` and `typeof()` return `"class"` when passed a class object.

#### Class Instances

Class instances are a new type of value in the VM.  They are similar but not quite the same as tables.  They have no array part, for instance.

`pairs`, `ipairs` , `getmetatable`, and `setmetatable` do not work on class instances.  They also cannot be iterated over with the generic `for` loop. (unless the class implements `__iter`)

Reading or writing a nonexistent class property raises an exception.  This makes it easy to disambiguate between a nonexistent property and a property whose value is nil.

The builtin `type()` and `typeof()` functions return `"object"` for any class instance.  We chose this over having them return the class name because class names do not have to be globally unique (they must only unique within a single module) and because we do not want to make it possible for classes to impersonate other types.

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
    isinstance: (o: unknown, C: class) -> boolean,
    classof: (o: unknown) -> class?,
}
```

This library also serves as an obvious extension point for future features like reflection.

The function `class.isinstance(o, Class)` returns `true` if the object `o` is an instance of `Class`.  At runtime, it raises an exception if the second argument is not a class object.  If the first argument is not a class instance, `class.isinstance` returns false.  (eg `class.isinstance(5, MyClass)`)

### Type System

Class definitions also introduce a new type to the type environment.

Unlike tables, which are structurally typed, class types are nominal.  Two different classes with identical fields are treated as distinct types.

Inferring the types of class fields is fraught with difficulty, so un-annotated fields are given the type `any`.

The type introduced by a class definition is available anywhere in the source file.

The `class.isinstance` function participates in refinement:

```luau
function foo(p: unknown)
    if class.isinstance(p, Point) then
        return {p.x, p.y} -- no error here
    end
end
```

Each class object is a singleton instance of an unnamed type.  If needed, it is easy to access via `typeof(TheClass)`.  Class object types are all subtypes of the top `class` type.  We choose this name to make it clear that it is not the top type of class instances.

### Semantics

Class definitions are Luau statements just like function definitions.

The action of a class definition statement is to allocate the class object, define its functions and properties, and freeze it.  Consequently, a class cannot be instantiated before this statement is executed.

We do, however, *hoist* the class identifier's binding to the top of the script so that it can be referred to within functions or classes that lexically appear before the class definition.  This makes it easy and straightforward for developers to write classes or functions that mutually refer to one another.

Static analysis also considers the class's type to be global to the whole module so that it can appear in any type annotation anywhere in the script.

An example:

```luau
-- illegal: MyClass is not yet defined
local a = MyClass {}

-- OK: MyClass can appear in any type annotation anywhere
function use(c: MyClass)
end

function create()
    -- OK as long as this function is invoked after the class definition statement
    return MyClass {}
end

-- We can't statically catch this in the general case, but this will fail at runtime!
create()

class MyClass
end

local b = MyClass {} -- OK
local c = create() -- OK
```

Because class definition is a statement, class methods can capture upvalues just like ordinary functions do.

```luau
local globalCount = 0

class Counter
    public count: number

    function new()
        local count = globalCount
        globalCount += 1
        return Counter {count=count}
    end
end
```

### Out of scope for now

#### Private fields, const fields

These are things we want to do, but integrating them with the existing structural type system is surprisingly tricky and will be tackled in separate RFCs.

#### Generic classes

We're very excited to support generic classes and plan to introduce a fresh RFC to deal with them specifically.

#### Inheritance

We're still evaluating whether or not implementation inheritance is something we want to support.

Method inlining is something we'd like to try that is greatly complicated by inheritance.

Also, frankly, its worth as a programming technique is controversial: the [Fragile Base Class Problem](https://en.wikipedia.org/wiki/Fragile_base_class) can cause significant harm to a project.

Lastly, Luau easily supports interface inheritance through its structural type system, so inheritance is judged to be lower priority.

## Drawbacks

This is a really big feature that has lots of moving parts!

We need to introduce multiple new contextual keywords: `class` and `public` to start and `private` later.  We also introduce at least one new top type `class`. (we probably also need a corresponding `object` top type for class instances)

Allowing code to grab unbound method references (ie `local m = o.someMethod`) seems risky because it opens the doorway to a lot of difficult-to-optimize dynamism, but it also makes a bunch of nice things like `pcall` work exactly the way developers expect.  We're making the bet here that this does not materially affect our ability to optimize more mundane attribute access or method calls.

The word `class` is doing triple duty under this RFC: It is a contextual keyword, the name of a top-level library, and the name of the top type for class objects.

`class` is somewhat awkward as a type name.

Object oriented codebases tend to have far more cyclic dependencies between modules because every piece of data is also coupled to a whole bunch of functions that operate on that data.  We are probably going to have to work out a way to relax the restrictions on cyclic module imports.

## Alternatives

[Arseny's record proposal](https://github.com/luau-lang/luau/blob/7f790d3910bbfc2adf007da3551b0a13e42ebb7a/rfcs/records.md).  This proposal is really quite similar, but looks a bit more familiar to users coming from other languages and affords the development of features like private fields.

[Shared Self Types](https://github.com/luau-lang/rfcs/blob/master/docs/shared-self-types.md).  This proposal was intended to shore up table type inference in the case that the code was written in an OO style, but after significant work, it doesn't actually work all that well in practice.  The resulting system was very brittle and tricky to work with.  Trickier, in fact, than the pattern that developers are already writing today.

### Field syntax

We considered a number of possibilities for field syntax before settling on `public` and `private`.

Type annotations must be optional, so the syntax we choose must work well in that situation.

We considered using the existing `local` keyword for class fields, but judged it to be a bridge too far: Fields are not locals!  They do not inhabit the stack.

We also considered using no keyword at all and judged that to be unacceptable in the case that a field also has no type annotation.  A single bare identifier on a line all by itself looks too weird and might make the grammar difficult to change later on.

So there must be a keyword and it cannot be `local`.  Other keywords we considered were `var`, `let`, and `field`.

```luau
class Test
    local foo

    public foo
    public foo2: number

    private local foo2: number
    private foo3: number

    var bar
    var bar2: number

    field bar
    field bar2: number

    quux
    quux2: number
end
```

`let` is a great keyword to introduce a local binding, but does not make a ton of sense in a class declaration. (especially a declaration where we’re not providing an initializer!)

`var` is pretty good and also makes sense as a token to introduce a local, but is largely redundant.

`field` is spiritually aligned with the syntax sensibilities that drove the development of Lua, but has no precedent in any other language.

`public` and `private` work pretty well from a parsing perspective, have no historical precedent in Lua, and also encode ideas that we definitely want classes to support.
