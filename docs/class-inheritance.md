# Implementation Inheritance for Classes

## Summary

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

local dog = Animal { species = "Canis familiaris" }
local cat = Cat { species = "Felis catus", meowMult = 3 }
```

## Motivation

Other languages with support for object-oriented programming support inheritance in some way. We (the Luau team) have even written libraries which would benefit from having classes with nominal inheritance. Inheritance also makes it easier to write more modular code and provides another guidepost for programmers coming from other languages. Additionally, most Lua and Luau libraries which provide object oriented programming affordances make implementation inheritance available in some way.

## Design

### Syntax

We introduce two new contextual keywords: `open class` and `extends`.

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

local threedpoint = ThreeDPoint.new({ x = 0, y = 0, z = 0 })
local erroneous = ThreeDPoint.new({ z = 1 }) -- Type error.  x and y are uninitialized.
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

#### Name resolution

Methods defined on base classes are accessible via the derived class.

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

### Runtime

#### Declaration Order

The original class RFC states that class names are hoisted and so can be used before the class declaration has been evaluated.  This is unchanged by this RFC, but carries with it a consequence that is important to call out:

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

#### Memory Layout

Classes have a fixed layout.  In the VM, the mapping from name to offset is stored with the class object, not with individual class instances.

In the presence of inheritance, we specify one additional rule: All inherited properties of subclasses exist in memory at the same offsets.  Properties of the derived class occur afterward.

This means that, if the VM knows that some object is a subtype of some class type, it can access a particular known field at a particular known offset.  That offset will be correct no matter what the concrete type is.

#### Library Changes

The `class` library's `isinstance` member will be extended to support subclasses by extending class objects so each contains a nullable pointer to its superclass.

```luau
class.isinstance(cat, Cat) -- true
class.isinstance(cat, Animal) -- true
```

Notably, `class.isinstance` says a little bit more with inheritance into the mix: When the refinement is negated, we will sometimes infer some types that are a bit more interesting than expected:

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

get_class(Cat{}) -- returns Cat
```

The type signature of `class.classof` is unchanged from the original class spec: `(unknown) -> class?`

### Type System

In general, if a subclass meets the restrictions described above, then the type of its instances subtypes nominally against the type of its superclass's instances. For example, the following typechecks:

```luau
function printSpecies(animal: Animal)
    print(animal.species)
end

printSpecies(cat)
```

However, class objects never have any subtyping relationship.  For example, the following will not typecheck:

```luau
function initialize(cls: typeof(Animal))
    return cls { species = "Homo sapiens" }
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

Let's revisit a modified version of the initial example, this time with annotations on `self` describing what its type might theoretically be (annotations on the `self` parameter are actually syntax errors).

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

We said previously that methods that are redefined in subclasses must subtype the superclass's version. The subtype test for `__tostring` (with the annotations we've added) goes as follows:

- `(self: Cat, name: string?) -> string <: (self: Animal, name: string) -> string`
- `(self: Animal, name: string) <: (self: Cat, name: string?)` (method arguments are subtyped contravariantly)
- `(self: Animal) <: (self: Cat)` (we subtype corresponding members of type packs)
- `Animal </: Cat` Class types are always subtyped nominally. We define that superclasses do not subtype their subclasses, so this step fails.

To allow subclasses to override methods and have them typecheck, we add a special case to the type checker for overridden methods. More specifically, if we are subtyping two class methods where the first argument is named `self` for both, we skip subtying the first argument. Then, the subtyping test for `__tostring` proceeds as follows:

- `(self<Cat>, name: string?) -> string <: (self<Animal>, name: string) -> string`
- `(name: string) <: (name: string?)` (we skip `self` and subtype the other method arguments contravariantly as normal)
- `string <: string?` (this holds because the set of values represented by `string` is contained in the set of values represented by `string`)

We acknowledge the lack of a `self` type, which is necessary for methods that need to refer to the current class type even for subclasses. (eg to write a `function clone(self): self ... end`)  This can be proposed in another RFC.

## Drawbacks

We introduce even more keywords: `open` and `extends`.

Adding implementation inheritance reduces optimization opportunities involving class method inlining and dispatch. For example, consider the following:

```luau
function callToString(a: Animal)
    return a.__tostring()
end
```

Since `Animal` is `open`, we can no longer statically determine which method to dispatch within the body of `callToString`.

## Alternatives

We consider only single implementation inheritance to avoid the diamond problem. This is the same approach that Java takes.

We may consider multiple interface inheritance in the future.

We introduce constructors for classes in a separate RFC, which simplifies the picture for instantiating subclasses.

Instead of offering an `open` keyword to allow inheritance, we could follow in Java's footsteps by offering `final` instead.  Our assessment is that opt-in is the correct default.

We considered a `super` keyword, but found the justification for it to be very weak.  The developer can just write the actual name of the base class.  Additionally, we would have to answer some very awkward questions about what happens when the script defines another symbol named `super`. (which seems like a pretty good variable name!)
