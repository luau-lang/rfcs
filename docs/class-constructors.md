# Constructors

## Summary

Add user-definable constructors to Luau classes.

## Motivation

Constructing a fresh value of a class can be complex.  We have a bunch of constraints:

1. If a field has a type annotation, we need to provide a guarantee that the field always conforms to that type.
2. Many classes have complex invariants that the author would like to protect above and beyond the types of the fields.  Constructing an instance of the class might require doing arbitrary work.
3. Any initialization logic has to work the same way whether or not the actual concrete type of the object is that of the class or if it's some subclass.  In other words, the initialization logic doesn't know the final concrete type of the object!

Factory functions were initially envisioned as the implementation story for the above, but they fall short.

Consider this hypothetical example:

```luau
class BasePoint
    public x: number
    public y: number

    function create(): BasePoint
        return BasePoint.new { x=0, y=0 }
    end
end

class DerivedPoint extends BasePoint
    public name: string

    function create(): DerivedPoint
        -- We're stuck!  We cannot implement this function in terms of
        -- BasePoint.create() because it always allocates a fresh object.
    end
end
```

Implementation inheritance requires some mechanism by which a class constructor can do the work only to initialize its part of the object for some class that is at some indeterminate point in an inheritance hierarchy.

This is a very well-known problem.  The well-known solution is constructors.

## Design

All class instances are created by calling a built-in static method `.new()` on the class.  The method name `new()` is reserved by the language.  It is a parse error to attempt to define it.

Let `T` be a class object.  When `T.new(...args)` is invoked with any arguments, the following happens:

1. A fresh, uninitialized instance of `T` is allocated.  We'll call it `t` here.  All of its fields are initially `nil` irrespective of any type annotations.
2. `T.__init(t, ...args)` is invoked.  Any values returned by `__init` are ignored.
3. `t` is produced as the result of the expression

A class can invoke its parent constructor directly:

```luau
class A extends B
    public x: number

    function __init(self, x, y)
        B.__init(self, x)
        self.x = x * y
    end
end
```

### The Default Constructor

If a class does not explicitly define a constructor, it is given a default constructor.  The default constructor takes an optional mapping from property name to value and initializes the newly-created class instance with those properties.  If no argument or `nil` are passed, the default constructor simply initializes all class properties to `nil`.

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

### Rules of Constructors

In order to make uninitialized data unobservable, strict mode requires that constructors initialize their fields before use.  Additionally, `self` itself must not be passed or retained until all fields are initialized.  This is tricky to do in the general case, so analysis is limited to requiring the following:

* A constructor may not read any field of `self` before that field has been assigned to.
* A constructor must unconditionally initialize all of its fields before it can pass `self` to any function.
* Calls to base-class constructors are optional, but are recognized as a proper way to initialize all base class fields.  A derived constructor may decline to invoke the base class constructor and instead initialize all base class fields directly.
* Calls to base-class constructors are of course exempt from the rule surrounding use of `self`.  eg `BaseClass.__init(self, args)`
* This restriction also includes all method calls (eg `self:something()`) and operations that would cause an overloaded operator to be invoked.
* For v1, type inference does not attempt to track fields that are initialized within conditional constructs like `if` statements or loops.  Fields must be initialized directly at the function scope.
* Fields whose types are supertypes of `nil` are exempt from this requirement and are always considered to have been implicitly initialized with `nil`.  Explicit initialization of such fields is of course permitted.
* Additionally, any subexpression where `self` is explicitly cast is exempt from this. (eg `foo(self :: any)` or even `foo(self :: FooType)`)  We offer this as an intentional way for a developer to override the type system if they need to.

Once the base class has been called and all fields are initialized, constructors can do anything.

These rules are all enforced only by type checking.  When the program is run, uninitialized fields will have the value `nil` no matter what their types might say.

Class constructors inherit just like any other method.  This can be confusing and subverts our requirement that all fields be initialized, so both nonstrict and strict mode will warn if a class does not define an explicit constructor but inherits from one that does.

## Drawbacks

### Field Initialization Cannot be Guaranteed

We only check field initialization statically and only in strict mode.  This means that our runtime still has to be able to cope with fields that have been left uninitialized.

Some developers will feel inconvenienced by the restrictions on uninitialized class fields.  The current rules do not, for instance, permit a developer to write a helper function that partially (or completely) initializes a new class instance.  They must initialize it in-line.

Because base class constructors can potentially use `self` in any way once they have finished initializing fields, we acknowledge a loophole where uninitialized fields can be witnessed:

```luau
class BaseClass
    public base_field: string
    function __init(self)
        self.base_field = "My field"
        print(tostring(self))
    end
end

class ChildClass extends BaseClass
    public child_field: number

    function __init(self, value)
        BaseClass.__init(self)
        self.child_field = value
    end

    function __tostring(self)
        return `ChildClass \{ child_field = {self.child_field} \}`
    end
end

local cc = ChildClass.new(15) -- prints ChildClass { child_field = nil }
```

This is unfortunate, but any fix would require unacceptable restrictions or complexity.

### `.new()`

The choice to construct instances via a `.new()` static function is motivated by existing Lua libraries that offer ersatz objects.  Taking the name from developers is awkward, but in exchange, typical code using classes looks consistent with preexisting Lua libraries.

## Alternatives

Something we considered was offering field initialization a-la C++ to the constructor syntax.

```c++
struct S {
    int x;
    string y;
    bool z;

    S()
        : x(1)
        , y("Hello World!")
        , z(true)
    {}
};
```

This would offer a much more straightforward way to communicate the initialization requirements, but classes already introduce quite a lot of new syntax.  We decided that this was a bridge too far.

We considered using function calls as the syntax for construction (ie `local mc = MyClass()` instead of `local mc = MyClass.new()`).  This would have the advantage of keeping the method name `.new()` free for developer use, but this makes all classes use a different API from virtually all existing Lua code.

We also took a look at what it would be like for developers to be able to override `new()`, but it requires even more corner cases and caveats.  We judged that it was better in this case to have a simple, consistent rule.
