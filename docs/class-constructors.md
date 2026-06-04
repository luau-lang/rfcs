# Constructors

## Summary

Add user-definable constructors to Luau classes.

## Motivation

The initial class RFC specifies that all classes can be constructed by invoking the class object as a function with a mapping from fields to values as its sole argument.

This is great for "plain old data" classes and easily allows users to write their own static `.new()` method if they have more exotic construction requirements, but this falls apart in the face of inheritance because a `.new()` factory necessarily couples the actual class instance construction with the field initialization.

Concretely:

```luau
class BasePoint
    public x: number
    public y: number

    function new(): BasePoint
        return BasePoint { x=0, y=0 }
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

## Design

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

If a class defines an `__init` function, it is understood to be a constructor.  Classes with constructors follow different rules.  We define class construction as follows:

Classes that have constructors cannot be constructed via `T()` syntax.  Instead, the static method `.new` must be invoked.  Classes that do not define constructors are still initialized via `T {...}` syntax.

Let `T` be a class object that defines a constructor.  When `T.new(...args)` is invoked with any arguments, the following happens:

1. A fresh, uninitialized instance of `T` is allocated.  We'll call it `t` here.  All of its fields are initially `nil` irrespective of any type annotations.  
2. `T.__init(t, ...args)` is invoked.  
3. `t` is produced as the result of the expression

Classes are forbidden from expressly defining a `.new` method.  It is reserved by the language.

### Rules of Constructors

In order to make uninitialized data unobservable, constructors are required to follow some strict rules:

* The first argument of a constructor must be `self`.  
* A class must define a constructor if its base class defines one  
* If the base class defines a constructor, the class constructor must invoke it before reading or writing to `self` or any of its properties  
* A constructor must unconditionally initialize all of its fields before it can pass `self` to any function.  
    * The delegating call to the base class constructor is of course exempt from this.  `BaseClass.__init(self, args)`  
    * This restriction also includes all method calls (eg `self:something()`)  
    * For v1, type inference does not attempt to track fields that are initialized within conditional constructs like `if` statements or loops.  Fields must be initialized directly at the function scope.  
    * As a special exception, fields whose types are supertypes of `nil` are exempt from this requirement and are always considered to have been implicitly initialized with `nil`.  Explicit initialization of such fields is of course permitted.  
    * Additionally, any subexpression where `self` is explicitly cast is exempt from this. (eg `foo(self :: any)` or even `foo(self :: FooType)`)  We offer this as an intentional way for a developer to override the type system if they need to.  
* A constructor must not refer to any field before it has been initialized.

Once the base class has been called and all fields are initialized, constructors can do anything.

These rules are all enforced only by type checking.  When the program is run, uninitialized fields will have the value `nil` no matter what their types might say.

## Drawbacks

Constructors add more complexity to the language.  Some classes can be initialized via `T {x=x, y=y}` syntax and others must be initialized with different kinds of arguments.

We intentionally only check that fields are initialized in type checking.  This means that our runtime still has to be able to cope with fields that have been left uninitialized.

Some developers will feel inconvenienced by the restrictions on uninitialized class fields.  The current rules do not, for instance, permit a developer to write a helper function that partially (or completely) initializes a new class instance.

## Alternatives

We could follow in Python's footsteps and do away with the default `T{}` constructor, but this means that developers have to write a lot of dull code in the typical "plain old data" case:

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

We could make field initialization more logical by adding C++-style initializer syntax.  We're already adding a lot of syntax and I don't think it's worth it for us.

The choice to construct instances via a `.new` static function is motivated by Lua libraries that effect objects.  We sacrifice the ability for classes to define their own `.new` method but in exchange, code using classes looks more consistent with what came before.
