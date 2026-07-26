# Amended Classes (Draft)

## Summary

```luau
open class Animal as Ani
    public species: string

    function Ani:__post_init() -- Inherited and implicitly adds self
        self.species = string.lower(self.species)
    end
    function Ani:__tostring() 
        return `I am an animal of species {self.species}.`
    end

    function Ani.live() -- Inherited, but behaves like a static function
        return "I am alive"
    end
    local function new(name: string) -- Behaves like a static function, but overshadowed. Self is not automatically applied
        return Ani { species = name }
end

class Cat extends Animal as A
    public meowMult: number = 1
    public owner: string? = nil

    function Cat:__post_init()
      assert(not self.owner or self.owner ~= "me", "I do not like cats")
      self.species += " catus"
    end
    function Cat:__tostring(): string
        return `{A.__tostring(self)} and can meow with {self.MeowMult}x strength.`
    end

    function Cat:meow(): string
        return string.rep("meow\n", self.meowMult)
    end

    function Cat.fromAnimal(animal: A): typeof(Cat)
      return Cat { species = animal.species }
    end
    function Cat.new(species: string, meowMult: number, owner: string): typeof(Cat)
      local newCat = Cat.fromAnimal(A.new(species)) -- new shadows Animal's new, not overloads it

      newCat.meowMult = meowMult
      newCat.owner = owner

      return newCat
    end 
end

local dog: typeof(Animal) = Animal { species = "Canis familiaris" } -- Canis familiaris becomes lowercase
local worm: typeof(Animal) = Animal({ species = "Lumbricus terrestris" }) -- Valid syntax in accordance to existing luau grammar rules
local fish: typeof(Animal) = A {species = "Flippious flopmon" } -- Errors, as aliases are scoped to the class they are used in

local bird: typeof(Animal) = Animal.new("segalius flapmon")
local tabby: typeof(Cat) = Cat.new("tabiluis") -- Errors, as Cat's `new` is different from Animal's.
local tiger: typeof(Cat) = Cat.new("tigerious", 1, "Elon Musk")

local cat: typeof(Cat) = Cat { species = "Felis", meowMult = 3 } -- species becomes "felis catus"
local lion: typeof(Cat) = Cat { species = "Lie On", owner = "you" } -- meowMult is initialized with the specifed default of 1
```

## Motivation

Classes are currently being rediscussed due to holes and other design flaws in their design when initially merged. This draft RFC aims to address and propose new compromises on some of the issues raised by both the greater community and the official class redesign PRs.
As of writing, this design reference builds upon the 3 Classes amendments initially suggested by the luau team (PRs #210, #229, and #230), as the changes proposed are not large enough to justify rewriting everything from scratch. As such, anything not addressed here shares the decision initially decided in those PRs. When the draft PR is ready, the details mentioned in those PRs will be merged into this PR or split into 3 new ones.
For now, this amended class design only aims to implement QoL features needed for base-class syntax, class inheritance, and default constructors.

Not all of these ideas will make it to the final RFC, if any at all. Consider this document to be a brain dump for now

## Design

### Class Field Default Values
Class fields can take an optional default parameter via regular assignment syntax
```
public thing: type = defaultValue
public vagueThing = defaultValue
```
Fields with similar properties can also be batch declared like so:
```luau
public thing1: type, thing2: type = defaultValue
public vagueThing1, vagueThing2 = defaultValue
public Foo: type, vagueFoo = defaultValue
```
This is still valid, and functions largely the same as the initial classes RFC:
```
public thing: type
public vagueThing
public Foo: type, Bar: type
public vagueFoo, vagueBar
public vagueBaz, Baz: type
```
However, it is not safe to use these fields in non-conditional reads without first checking if it has been assigned a value. Code that does not do this will not run in strict mode

Finally, runtime variables and functions are allowed in field defaults. If the default expression has side effects or cyclic dependancies, that will be the user's job to handle unless there is a viable way to check for cyclic side effects in compiletime. 
Erroring default expressions propagate the error through class initialization as well.
Runtime functions that determine field defaults do not have access to the instance they are used to create. 

### self Keyword and Static Methods/Fields

Currently, `self` is an opt-out magic keyword that is automatically applied to class functions. While convenient, it is unintuitive for people learning the language or transitioning from metamethod classes.
Instead, class methods should continue to follow Lua's `.` and `:` syntax rules:
```luau
class Foo
  function Foo:thing1() end -- self is implicitly usable 
  function Foo.thing2(self: Foo) end -- self is usable and functionally identical to thing1
  function Foo.thing3(self) end -- self is usable but is typed to any/unknown
  function Foo.thing4() end -- self is unusable unless defined as a local variable
  function thing5() end -- functions identically to thing4
end
```
Note that the exact semantic rules for functions like `thing5` have yet to be fully deliberated on. See `Alternatives` for details.

For now, Class methods with `.` prefix or no prefix at all function identically to static functions in other languages.

Fields can also be declared through `public Class.fieldName`, which functions identically to static fields in other languages
Fields cannot be declared through `public Class:fieldName` due to ambiguity concerns (`:` is supposed to function as syntax sugar for self)

### `as` Keyword and `super()` Inheritance

Note: `as` keyword will likely move to a separate RFC due to its described behavior being extremely out of scope implementation-wise compared to the problem that it solves. However, the point of not introducing `super` into Luau still stands.

The current inheritance design intentionally excludes `super()` in favor of directly referencing the parent class. This is not a bad alternative, but it comes at the cost of coupling the child class directly to its parent. This makes class inheritance fragile and creates a major annoyance when the user needs to refactor a base class with multiple children spread across many files 
On the other hand, `super()` is a poorly named and incredibly inflexible keyword whose use is not inherently obvious to the vast majority of new programmers until they go out of their way to learn how it works. 
Many others have made suggestions on how to bring super-adjacent functionality, but most try to port the keyword too literally and introduce all the problems previously discussed as well as some additional implementation/design hurdles

To sidestep such issues, class declarations should gain an additional `as` keyword:
```luau
class Foo extends Bar as Baz
```
The `as` keyword allows the declaration that immediately follows it to act as an alias for the declaration behind it. 
Both declarations that pertain to `as` must both exist and be user-definable, and neither declaration can contain reserved keywords. For example:
```luau
class Foo extends Bar as self/if/continue/break...
Bar as class Foo extends Baz
class Foo extends end as Bar
class Foo extends as as as
class Foo extends continue as
```
Are all banned uses of `as` that error upon use.

In future RFCs, `as` can flexibly scale with potential future features of classes, such as composition or multiple inheritance as teased in the original RFC (I do not recommend implementing multiple inheritance):
```luau
class Foo composes A as apple, B as bus, C as cat
end
-- Or grouping syntax:
class Foo composes {A, B, C} as BundleA, D as BundleB
end
```

Additionally, `as` aliases (the declaration to the right of `as`) are scoped to the class that uses it and follow normal scoping procedure:
```luau
local Baz = 42
-- Use of Baz returns 42
class Foo extends Bar as Baz
  -- Use of Baz returns Bar
end
-- Use of Baz returns 42 again
class Foo2 extends Bar2 as Baz
  -- Use of Baz returns Bar2
  function Baz() -- Luau lints with localShadow warning 
    local Baz = 5
    -- Use of Baz returns 5
  end
  function useBaz()
    -- Use of Baz returns Foo2's Baz() and is no longer an alias of Bar2
  end
end
-- Use of Baz returns 42 again
```

Under these rules, `as` can be syntax sugar for class name as well:
```
class Foo as F extends Bar as B
  public F.staticField -- Valid, as F is shorthand for Foo
  public Fizzbuzz: number = 42

  function F:__toString() -- Also valid
    self.Fizzbuzz += 1
    return B.__toString(self) -- Calls Bar
  end
end
```

Unfortunately, aliases through `as` do not stop you from mixing and matching the original declaration with its corresponding alias to your own detriment:
```
class Foo as F extends Bar as B
  public F.staticField
  public Foo.anotherStaticField
  public Foo.Fizzbuzz: number = 42

  function F:__toString() 
    return Bar.__toString(self)
  end
  function Foo:__add()
    return B.__add(self)
  end
end
```
Such use is highly discouraged and has no functional use

Finally, `as` declarations may also be used in types as long as no existing type alias. If both an `as` alias used as a type cast and a type alias share the same name, luau errors:
```luau
type thing = string

class Foo as F extends FizzBuzz as FB end -- valid

class Bar as B extends Fizzbuzz as thing end -- invalid, but Luau doesn't error until `thing` is used as a type cast

class Baz as Baaaz extends Fizzbuzz as thing

function Baaaz.FromFizzBuzz(fb: thing) end -- invalid, and luau gives a type error here due to type redefinition via alias
end
```

Any use of `as` outside class declarations is neither in scope of the RFC nor recommended practice in most (but not all) cases.

### Static Shadowing and Opt-Out Inheritance

The original class syntax supported factory functions as such:
```luau
class Point
  function fromDirection(theta, length): Point
    return Point {
      x = length * math.cos(theta),
      y = length * math.sin(theta),
    }
    end
end
```
However, factory functions of this sort become a hindrance in inheritance:

```luau
open class Point
  public x: number = 0
  public y: number = 0

  function fromDirection(theta, length): Point
    return Point {
      x = length * math.cos(theta),
      y = length * math.sin(theta),
    }
  end
end

class Point3D extends Point
  public z: number
end
-- later in code
local my3DPoint = Point3D.fromDirection(60, 30) -- extremely misleading method that technically works!
```
Aside from the inability to overload in certain situations, there is also a chance that the end user doesn't want their children using certain fields. While this can be partially solved through the planned `private` keyword, it's too restrictive and doesn't account for situations like the one described above, where `fromDirection` still needs to be accessible for enduser use

There are many possible solutions to this problem, but not many are desirable due to how inheritance is defined in the original design:
> However, the method declared in the subclass must be a subtype of the method in the superclass. That is, a child class can override a parent class method with a function that is more permissive than that of the base class method, but not less.

Interestingly, there seems to be an exception carved out specifically for class objects:
> However, class objects never have any subtyping relationship
If true, then theoretically static/class-level data could safely allow some form of shadowing: 
```
open class Point
    function Point.fromDirection(theta: number, length: number): Point
        return Point {
            x = length * math.cos(theta),
            y = length * math.sin(theta),
        }
    end
end

class Point3D extends Point
    function Point3D.fromDirection(theta: number, length: number, phi: number): Point3D
        return Point3D {
            x = length * math.cos(theta) * math.cos(phi),
            y = length * math.cos(theta) * math.sin(phi),
            z = length * math.sin(theta)
        }
    end
end
-- later in code
local myPoint = Point.fromDirection(60, 30) -- Returns a point
local my3DPoint = Point3D.fromDirection(60, 30) -- Errors, as `fromDirection` doesn't have enough arguments
```
Since shadowing is different from overloading, referring to the shadowed parent static fields and methods from a child class can still be done:
```luau
class Point3D extends Point as Parent -- see the `as` keyword section if this confuses you
  public z: number

  -- Use `super` or otherwise directly call the class parent
  function Point3D.fromDirection(theta, length)
    local point = Parent.fromDirection(theta, length)
    return Point3D { x = point.x, y = point.y, z = math.huge }
  end
```
This may also open the way for static fields and functions to completely "opt out" of inheritance altogether.
However, this still requires redefining some rules about static methods and taking some tradeoffs, since it could create unsavory situations:
```luau
local point: Point = Point3D {z = 10} -- Because Point3D is a child of Point, this is technically type legal

local newPoint = point.fromDirection(theta, length) -- However, due to shadowing, this causes the type checker to lie
```
There are potentially multiple ways to fix this while still allowing for static shadowing, but none are small changes. As things currently stand, the type system does not have enough information about classes to handle such situations. A section dedicated to explaining this can be found near the bottom of the page.
For now, the "how" can be decided in future revisions of this RFC if it is deemed worthy of the effort, as it likely requires new syntax/grammar.

### Default Constructors

Potentially the most controversial section of the proposed classes RFC, the default class constructor terminology has many issues with no clear "correct" answer that does not come accompanied with varying levels of unintuitive compiler "magic", drawbacks, and branching behavior

While this solution tries to minimize such issues, it is ultimately no different. This solution attempts to rectify default constructor issues by forgoing runtime lifecycles and hooks to initialization altogether in favor of a constructor-like syntax that is controlled by luau itself:
```luau
class Foo
  public Bar: number
  public Baz: string

  function Foo:__post_init()
    print
  end
end
-- Later in code
local loreIspum = Foo { Bar = 42, Baz = "Fizz buzz"}
```
When `Foo` is directly called (via `Foo({})` or `Foo {}`), luau calls an internal `__init` function that theoretically behaves like so pre-optimization:
```luau
-- Note that this is pseudo code
function Foo.__init(args)
  local instance = Foo 
  local parent = instance.Parent:.__init(args)

  for name, data in class.getInheritableData(parent) do
    instance[name] = data
  end

  for field in class.getPublicFields(instance) do
    if args[field] == nil and not field.hasDefault and luau.isStrictMode then
      error(`Intitalization is missing required field {field} in class {instance}`)
    elseif args[field] then
      instance[field] = args[field]
    end
  end
  -- doesn't return anything because it still needs to go through its __post_init pass
end
```
`__init` is not publicly accessible and only assigns public variables. No `__post_init` runs until every `__init` in the inheritance hierarchy runs first.
Private field initialization (once implemented) and other tasks are assigned in `__post_init`, which is publicly reassignable by the runtime and end users.
`__post_init` is not required to be specified by the end user. If unspecified, that class' `__post_init` is an empty function

By default, __post_init runs in order of oldest to youngest inheritance parent. If possible, the fields inside each class are initialized from top to bottom. This RFC draft is currently unsure if endusers should be allowed to reorder or even call inheritance `__post_init`s due to the implementation headache and design questions they introduce. Deferring such logic to a future RFC may make sense.

Note that this only applies to default constructors. Custom constructors with the following syntax are still allowed:
```luau
class Point
  public x, y
  
  function Point:__post_init()
    -- validation code here...
  end
  
  function Point.new(x: number, y: number)
    return Point { x = x, y = y }
  end
end
```
## Drawbacks

### Class Field Default Values
- Introduces more implementation work than in the original RFC
- Default values always have the inherent problem of being enduser unfriendly due to the user needing to look inside the class itself to see what happens when a default variable is called
- Allowing runtime variables in default classes adds extra implementation baggage/questions to look out for, and allowing function calling can result in foul code hygiene if wielded by the wrong person (through side effects) 
- Thanks to the proposed Default Constructors redesign, private default values are an open design question

### self Keyword and Static Methods/Fields
- While the `as` keyword helps, it is still more boilerplate that the enduser has to use to make function methods. Since `self` is so common in OOP and classes, explicit clarity that people may not value comes at a high cost to the enduser 
- Allowing "unnamed" functions/fields that have different defaults (unnamed fields default as instance fields while unnamed methods default as static functions) can be seen as weird and inconsistent to the enduser or someone new to Luau

### `as` Keyword and `super()` Inheritance
- All the drawbacks that come with introducing new keywords ("bloats" the luau grammar, breaks existing user code that used `as` for variable names, etc) with small initial gain (`as` is only proposed for use in class declarations)
- Its central use case (ensuring stronger and easily maintainable inheritance) is weak compared to other keywords. While it is an essential QoL addition for many, it is not vital to the existence of classes in Luau
- The ability to mix Alias and non-alias names in the same class is a very easy way to create ambiguous and hard-to-read code

### Static Shadowing and Opt-Out Inheritance
- Such syntax might pose consistency issues, as it is not easy to explain/understand/intuit why shadowing works for static data, but not instance-specific data. 
- Requires class-specific type system enhancements to aid with inheritance refinement
### Default Constructors
- Users are shackled to the predetermined whims of the Luau compiler, as they can no longer have fine-grain control over almost every aspect of class initialization. The impact may not be huge in the grand scheme of things (there is very little that you can't do in `__post_init` that you can do in `__init`), but it is real nonetheless
- Luau internally handling `__init` is still somewhat magical, and it is not obvious to the enduser what `__init` will be doing unless they investigate themselves
- Potentially very unreliable, as there is no set initialization order of public field initialization within any given class.
- Strong rules governing `__post_init` usage can meaningfully affect and hinder the use of classes by others (cannot do anything before initializing/post-initializing the parent class)
- Parent -> children ordering, while intuitive, can still be unideal in certain circumstances

## Alternatives

### Class Field Default Values
- Default and named parameter support erases the ergonomic benefits of default fields, and is a feature that can be widely used outside of classes. However, much of this RFC is built around the idea of default field parameters, and default and named parameter support for functions is its own implementation beast.
- Do nothing, and force all field initialization to happen in `__init` or `__post_init`. This allows for better code readability at the cost of more frustrating initialization assuming Default Constructors is implemented as described in this draft RFC. Otherwise, the cost of implementation is also passed to the class creator themselves, as they need to manually `__init` every variable themselves in 1 or 2 big functions

### self Keyword and Static Methods/Fields
- Introduce a `static`-adjacent keyword instead of relying on Lua philosophies. This will also contain all the problems associated with adding a new keyword to Luau with very little use outside of classes. A `static`-like keyword also must be stackable with other keywords, bringing about the Java problem again. The word `static` is also not inherently obvious about what it does. New devs will have to go out of their way to learn what it does
- Forgo the classname and only include the bare necessities (`.` and `:`). This works, but it is syntactically ugly and potentially easier to miss. People used to Lua's `.` and `:` notation may still write the classname regardless due to muscle memory. Outside that, there are no further obvious drawbacks.
- Continue with the original plan, and have `self` be a required positional argument implicitly added to every class method function that doesn't use `className.methodName` syntax. The impact has already been discussed in the design section for this change
- Do nothing and/or defer static and self logistics to a future RFC. However, Classes will be neutered until an equivalent is implemented

###  `as` Keyword and `super()` Inheritance
- Implement `super` as a keyword, class library method, built-in function, or by some other means. By doing so, Luau inherits the many problems that came with `super`:
  - Special enforcement rules must be made to ensure super is only used in classes
  - `super` in and of itself is an unclear keyword that is not intuitive until you investigate and see it in action
  - `super`'s "all or nothing" nature hampers Luau's ability to support potential future uses of classes that require support for multiple "parents" (like composition or inheritance)
  - Proposed `super` syntax in most of the ways described is ugly and/or annoying
- Do nothing, and force class children to directly couple code to their parent classes. This makes inheritance code more fragile and means that any change to a central base class will have lasting, somewhat unavoidable ripple effects across the rest of the tree. Unsure how problematic this truly will be.

### Opt-Out Inheritance for Methods/Fields
- Reuse `local` for static fields to opt out. However, this comes with many problems:
  - `local` is an extremely problematic keyword, almost to the same degree as using `local` to designate public/private classes. To people who built their mental model of `local` around a more literal interpretation, this is a jarring keyword to use for this reason
  - `local` can cause confusion when combined with field keywords like `local public x` or `local private x` if your mental model of `local` is something along the lines of "this is a private variable"
  - Starts a trend of Java-like keyword stacking, which will intimidate people wanting to learn Luau for its simplicity. Allowing this may set an unwanted precedent for future design decisions
- Create an exclusive "uninheritable" keyword instead of reusing `local`. Prone to the same drawbacks as `local`, but has the potential to be more/less confusing for end users depending on the name given
- Combine "uninheritable" functionality with static. All static functions are automatically ignored by inheritance. Will not introduce new keywords, but messes with users who want static class functions to be inheritable across all children (which has real usecases and doesn't follow static convention from other languages)
- Create a `noinherit` attribute. Will work, but requires implementation engineering for attribute use on fields as well as special-case validation work to ensure "uninheritable" is not used outside of classes. Likely not worth the technical debt/mess it would bring
- Reuse `open` for methods/fields. Also will work, but `open` is opt-in. Will likely cause more problems than it solves
- Functions that aren't specified with `.` or `:` are ignored in inheritance. Works conceptually, but it's easy for such syntax to be used accidentally and become an invisible bug due to a typo. 
- Indirectly solve the issue by implementing something akin to Python `classmethods` (which allow for use of parent class factories in children). This solves the example given in said section, but does not address other valid reasons for not wanting to inherit a particular field or method
- Do nothing, and force every public field and method to be inherited indiscriminately. The drawbacks of such have been described in-depth in the section itself

### Default Constructors
- By @TenebrisNoctua in #210: Allow C++-style inheritance as described [here](https://github.com/luau-lang/rfcs/pull/210#issuecomment-4632943714), and reintroduce `__init` as a usable/callable baseclass. More boilerplate, but also more power. You can choose which fields you like to implement, the order of their implementation, and the order of the initiation class functions themselves. It is also less magical than the proposed default constructor idea, as `__init`-less functions are constructed as a typed empty metatable instead of them going through the initialization algorithm behind the scenes. However, an empty, unititalized table poses type system design gaps (everything is `nil`, regardless of type information) and it loses the readability benefits of the current constructor method. It also loses the guarantee that every public field is initialized before side effects begin, potentially leading to weaker or harder-to-implement type enforcement. If going forward with this idea, it is highly recommended to strongly consider implementing default fields, default parameters, and named parameters for a better user experience
- By @InfraredGodYT in #216: Have a dedicated `constructor()` function type that handles `__init` as described [here](https://github.com/luau-lang/rfcs/pull/216#issue-4814448206). This is more honest in its functionality than `__init`, but results in more keywords and grammar added to Luau's lexicon (with all the aforementioned problems that come with it). Has similar boilerplate, lack of initialization guarantees, readability, and a magical default `__init` that must be expressed through special initialization syntax (`Cls {}` instead of `Cls()`)
- Continue with the original plan and face the downsides addressed in its PR comments (#210)
- Do nothing, and force explicit construction via factory functions. Still has all the problems faced with the previous alternatives

## More on Shadowable Static fields/methods 
Getting shadowable functions to work with the type system takes considerable work. The "easiest" implementation would be as follows:
1. Disallow instances to call static data, which makes `local newPoint = point.fromDirection(theta, length)` illegal and leaves this as the only legal syntax:
```luau
local pointCls: Point = Point
pointCls.someStaticVariable = newValue
local newPoint = pointCls.fromDirection(theta, length)
```
2. Ensure `class.classOf` returns the Class that made the instance, and not a fresh instance of a different class. That way, you are still able to edit class variables from an instance:
```luau
local recoveredClass = class.classOf(newPoint)
```
3. Give classes stable identities, so the type checker can refine `class.classOf` back into `Point`. There are a couple of ways this could happen:
```luau
assert(recoveredClass == Point) -- Option 1: allow classes returned by `classOf` to have equivalence
assert(class.isclass(recoveredClass, Point)) -- Option 2: introduce a stricter version of `class.isinstance` that only returns true if the class matches exactly
```
Option 1 could introduce problems with equivalence metamethods, so option 2 is likely the safer choice. 

By the end of the "easiest" solution, safely editing static methods without direct access to the initializer class looks like this:
```
function recreate(pnt: Point)
  local cls = class.classOf(pnt)

  if class.isClass(cls, Point) then
    -- use Point exclusive static data
  elseif class.isclass(cls, Point3D) then
    -- use Point3D exclusive static data
  else
    -- you are done for
  end 
end
```
A harder but more pragmatic approach to ensuring type safety is to introduce a compiler-owned magic "type function" like `ClassOf<T>` and require `class.classOf` to return `class.classOf<T>?` instead of the vague `class?`. However, the specifics of this and the runtime type refinement likely deserves its own RFC.

The bigger question is whether making massive updates to the type system and static behavior for this particular edge case (getting classes from instances) is worth it for the ability to support shadowing of static methods/fields. As a draft RFC, the goal is to put such ideas up for discussion to decide if such ideas are worthy of being translated into a proper PR. Opinions are much appreciated. 
