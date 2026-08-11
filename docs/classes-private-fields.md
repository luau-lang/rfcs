# Private Class Fields

## Summary

We adopt [JavaScript-style](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements) private class fields and methods: field and methods are declared private by prefixing them with `#`.

Throughout this PR, we use “properties” to mean the set of a class’s field and methods.

```
class Point
    local #x: number
    local #y: number

    function getX(self)
        return self.#x
    end

    function getY(self)
        return self.#y
    end

    function #computeMagnitude(self)
        return math.sqrt(self.#x ^ 2 + self.#y ^ 2)
    end

    function getMagnitude(self)
        return self:#computeMagnitude()
    end

    function __init(self, x, y)
        self.#x = x
        self.#y = y
    end
end
```

Private properties can only be accessed by the methods of the class in which they are defined. 

Private fields are not visible outside the class in which they are declared. As a result, instances of classes with private fields can only be directly constructed within the class where they are declared. If such a class is to be instantiated elsewhere, it must define a constructor.

Subclasses do not have access to their ancestors’ private properties. This means that private properties can be shadowed. A base class and a derived class can both have private properties `#x` that do not collide or interact in any way. A derived class can also define a public property `x` that shadows a base class's `#x` field.

## Motivation

An important feature for library authors is a way to hide implementation details and force users through a curated interface that is intentionally kept stable.

There are two ways to afford this in Luau today, and both of them have problems:

### Closures

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

### Convention

Secondly, library authors can use a naming convention (like `_prop`) to indicate data that is not part of the public interface and ask developers not to use those symbols.  This is more efficient because it allows method implementations to be shared between instances and so introduces way fewer closures, but tends not to work very well because users inevitably write code that depends on this state.

Roblox itself has used this technique and found itself forced to permanently preserve supposedly "private" properties.

## Design

Methods and fields can both be declared private by prefixing with `#`. A private field can only be accessed by methods of that class. Private methods can only be called by methods of that class. Static methods can also be private.

As is the case with public properties, using keywords as the names of private properties is forbidden.

To avoid complications involved with the notion of a private metamethod or metatable, declaring private property that begins with `__` is forbidden. Defining a private property named `#new` is also disallowed.

A class’s private property names cannot shadow any of its or any ancestor’s public properties. This simplifies parsing for a possible private default constructor (discussed below).

Fields will now be indicated with `local`, rather than `public`. We considered having no keyword at all for non-private fields, but the lack of required separators between fields makes it difficult to introduce contextual keywords in the future, such as for `const` fields.

Private fields are accessed similarly to the familiar `.` and `:` operators: `obj.#privateField` or `obj:#privateMethod()`. Using `.#` and `:#` outside of a class method body is a syntax error.

Accessing private fields is implemented via the new `GET/SETPRIVATEFIELD` (actual name TBD) instructions. They are emitted if and only if the bytecode compiler observes `.#` or `:#`, and will have roughly the following format:

| Slot | Content |
| :---- | :---- |
| A | Target/source register |
| B | Register containing class instance |
| C | Register containing expected class |
| AUX\[0:14\] | Private property index We will restrict the number of private instance properties and private static properties a class can have to 32,768 each |
| AUX\[15\] | Whether property is instance property or static property |

We restrict our usage of AUX to just 16 bits to leave room for potential future changes.

At runtime, `GET/SETPRIVATEFIELD` will perform a tag check to ensure that `B` is both a class object and an instance of the correct class. Since `.#` and `:#` can only occur inside class method bodies, the bytecode compiler can statically determine which class an instance should belong to. Additionally, hoisting means that the relevant `LuauClass*` will be present in a register, which the compiler can include in `C`.

To facilitate statically computing the index of private properties, we will add a `TValue` array to `LuauObject` that will hold the values of private fields, and another `TValue` array to `LuauClass` to hold the values of private static members (currently only methods). The indices into each array are known at compile time since classes cannot access the private properties of their ancestors.

Inside a method body, the private fields of any instance of the enclosing class (not just self) are available. This means in the following, the `__eq` metamethod will succeed if invoked on an instance of `Point` as `other`, and error at runtime if passed an argument of any other type.

```
class Point
    local #x: number
    local #y: number

    ...

    -- Note that no annotation is required for 'other' here
    function __eq(self: Point, other)
        return self.#x == other.#x and self.#y == other.#y
    end
end
```

Type inference can also be extended so that private field access in a method body applies an equality constraint between the object being accessed and the enclosing class. In the above example, `other` would be inferred to have type `Point`.

It is safe for private class properties to be shadowed, since child classes cannot access private fields and methods of their ancestors.

### Private Fields and Constructors

We have two overarching goals for how constructors should interact with private fields:

- Minimize how often instances can have uninitialized state  
- Don’t make behavior too complex to reason about.

To these ends, classes that have private properties must explicitly define a constructor.  The default constructor will raise a runtime exception when called.  This is necessary because the default constructor would otherwise need to initialize private fields--fields that users of the class shouldn't even be aware of\!

Note that this restriction interacts with the existing requirement that if a base class defines a constructor, then any derived class must also do so. Concretely, if a class inherits from a base class with private fields, then it must now define a constructor.

We considered making the default constructor private if the class has any private fields, but this runs into the same problem as a `private` keyword. (discussed in the Alternatives section) We also considered adding private constructors, defined via `#__init` and invoked via `#new`, but we opted against it, both to align with the second goal above and to limit the scope of this RFC.

## Drawbacks

Luau’s dynamism means that the runtime has to do some tag checks. Consider:

```
class OtherFoo
    local #x: number

    function __init(self, x) self.#x = x end
end

class Foo
    local #x: number

    function __init(self, x) self.#x = x end

    function printX(self)
        local y = if math.random() > 0.5 then self else OtherFoo.new(2)
        print(y.#x) -- requires a tag check
    end
end
```

It is impossible to statically determine what the type of `y` will be inside the print statement, so we necessarily need to do a tag check when we index into it at runtime. 

## Alternatives

### `private` keyword

One alternative we are considering is to use a conventional `private` keyword.

```
class Point
    private x: number
    private y: number

    function compare(self, other)
        return self.x == other.x and self.y == other.y
    end
end
```

In this example, the parameter `other` has no annotation, so we do not know if we are accessing a private field of a `Point` or just an ordinary key of a table.

When compiling this code, we observe that `other.y` occurs within a context where we *might* be asking for the private `y` field of another `Point` and so we emit a new instruction `GETPRIVATEFIELD`. (truename tbd)

`GETPRIVATEFIELD` checks the tag on `other` and if it is a `Point`, it pushes the private `y` field of that instance and skips the following instruction.  It is always followed by a `GETTABLEKS` instruction which is used as the fallback.

This incurs a little bit of interpreter overhead as the VM will need to walk the inheritance chain of `other` in order to decide what to do.  We may be able to mitigate this overhead somewhat by using an inline cache to hold the previous class pointer if the lookup was successful.

There is another ergonomic issue that may rear its head: A derived class can declare a public field that shadows a private field of a base class.  This can lead to an unfortunate ambiguity:

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
```

We would have to ask the developer to write `other['x']` if they specifically wish to access a public field named `x` within a class that also defines a private `x`.

### Secret Property Names

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

### Property rewriting

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

### Restrict private field access to `self`

We considered allowing private field access on only `self`, but this is quite restrictive.

### Module-specific `private` fields

One possibility was for `private` fields to be accessible from throughout the module in which they’re defined, but this permissiveness seems more appropriate for something like a `friend` access modifier, and the Luau VM would require significant reworking to make this work.

### Private constructors

We considered adding private constructors, defined via `#__init` and invoked via `#new`.

If a class defines a method `#__init` or has at least one private property, the class would instead have a builtin method `#new`, which could be used by other methods of the class.  Concretely:

```
class Point
    local #x: number
    local #y: number

    function zero(): Point
        return Point.#new { x = 0, y = 0}
    end
end
```

We've decided not to implement this at this time because such a private constructor can only be called by public methods of the class, so the encapsulation benefit here is pretty minor.  In order to offer more, we would have to introduce some mechanism for a symbol to be private to a larger organizational unit like a package.  We are opting not to open this particular can of worms just yet.