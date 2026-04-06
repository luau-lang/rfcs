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
* A construct with a fixed shape and a completely locked-down metatable will open up optimization opportunities that could improve performance.

## Design

### Syntax

Class definitions are a block construct.  They can only be written at the topmost scope.  `export class X` is allowed.

Defining two classes with the same name in the same module is forbidden.

Within a class block, two declarations are allowed: Fields and methods.

Fields are introduced with the new `public` keyword.

Methods are introduced with the familiar `function` keyword.  `public function f()` is also permitted.

If a method's first argument is named `self`, it can be invoked with the familiar `instance:method()` call syntax.  Type annotations on the `self` parameter are not allowed.

If a method accepts no arguments, or if its first argument is not named `self`, it can be invoked via `ClassObject.method()` syntax.  This is the same as "static methods" from other languages.

To create a new instance of a class, invoke it as if it were a function.  It accepts one argument: A table that describes the initial values of all its properties.  If more customization is desired, static factory functions (frequently named `new()` or `create()`) are an easy, familiar way to accomplish this.

Classes can define familiar Luau metamethods like `__add` and `__sub`.  They will work as one would expect.  `__index` and `__newindex` may not be defined.

#### Class Instances

Class instances are a new type of value in the VM.  They are similar but not quite the same as tables.  They have no array part, for instance.

`pairs`, `ipairs` , `getmetatable`, and `setmetatable` do not work on class instances.  They also cannot be iterated over with the generic `for` loop.

We introduce a new global function `instanceof(a, Class)` which returns `true` if the object `a` is an instance of `Class`.  `instanceof` raises an exception if the second argument is not a class object.  If the first argument is not a class instance, `instanceof` returns false.  (eg `instanceof(5, MyClass)`)

Reading or writing a nonexistent class property raises an exception.  This makes it easy to disambiguate between a nonexistent property and a property whose value is nil.

The builtin `type()` and `typeof()` functions return `"object"` for any class instance.  We chose this over having them return the class name because class names do not have to be globally unique (they must only unique within a single module) and because we do not want to make it possible for classes to impersonate other types.

```luau
class Cls end
local inst = Cls {}

type(Cls) == 'object' -- Class objects themselves behave like class instances
typeof(Cls) == 'object'

type(inst) == 'object'
typeof(inst) == 'object'
```

#### Class Objects

The action of evaluating a class definition statement introduces a *class object* in the module scope.  A class object is a value that serves as a factory for instances of the class and as a namespace for any functions that are defined on the class.

Class objects behave like class instances in most ways, but are always `const` and frozen.

Taking references to class methods via `ClassName.method` syntax is allowed so that classes can easily compose with existing popular APIs:

```luau
local n = pcall(SomeClass.getName, someClassInstance)
```

To construct an instance of a class, call the class object as though it were a function.  It accepts a single argument: a table that contains initial values for all the fields.

### Type System

Class definitions also introduce a new type to the type environment.

Unlike tables, which are structurally typed, class types are nominal.  Two different classes with identical fields are treated as distinct types.

Inferring the types of class fields is fraught with difficulty, so un-annotated fields are given the type `any`.

The type introduced by a class definition is available anywhere in the source file.

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

We need to introduce multiple new keywords: `class` and `public` to start and `private` later.

Allowing `ClassObject.someprop` seems risky because it opens the doorway to a lot of difficult-to-optimize dynamism, but it also makes a bunch of nice things like `pcall` work exactly the way developers expect.  We're making the bet here that this does not materially affect our ability to optimize more mundane attribute access or method calls.

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
