# Function Calls with Named Arguments

## Summary

This RFC proposes adding optional __named arguments__ to Luau function calls, allowing arguments to be specified by parameter name rather than position.

## Motivation

Function calls in Luau currently rely on positional arguments, which can make code hard to read when parameters are not self-describing. This is especially problematic for primitive types, such as numbers and booleans, which carry no semantic meaning outside their context.

Consider this example:

```luau
type FS = {
    read watch: (path: string, recursive: boolean) -> (),
} 

local fs: FS

fs.watch("/", true)
```

Here, the first argument is self-describing (in that for a filesystem function, you're expecting a path), but the second argument is ambiguous: it could be `recursive`, `blocking`, `follow_symlinks`, et al. If you don't read the function definition, you __can't__ know.

The solution proposed by this RFC is to use named arguments:

```luau
fs.watch("/", recursive=true)
```

For another example, consider this pattern, commonly seen in game development:

```luau
-- I'll omit the definition of `Car` to make the point clear

local car: Car

car.new(350.0, 275.0, john_doe)
```

Do the first two arguments refer to hit points, horsepower, maximum speed, or acceleration? Who is `john_doe`, the owner, or the driver?

Using named parameters makes it clear:

```luau
car.new(horsepower=350.0, hp=275.0, owner=john_doe)
```

A less common but still relevant issue is:

```luau
type Foo = (bar: Bar, qux: Qux, foobar: number) -> string
type Bar = () -> (...any)
type Qux = (quux: number, quxquux: number, foobar: number, ...any) -> string

local foo: Foo = function(bar: Bar, qux: Qux, foobar: number): string
    return qux(bar(), foobar=foobar)
end
```

This sort of pattern is typically seen in event handlers, and it is currently not possible to do without making extra unnecessary function calls.

To end the motivation section, here are a few functions from Roblox, a popular runtime for Luau, that could benefit from named arguments:

```luau
TweenInfo.new(time=2.0, delayTime=0.5, repeatCount=5, reverses=true)

workspace:Raycast(origin=Vector3.zero, direction=-10.0 * Vector3.yAxis)

TweenService:SmoothDamp(current=Vector3.zero, target=target, velocity=Vector3.yAxis, smoothTime=1.0, maxSpeed=10.0, dt=dt)
```

## Design

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this section are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### Syntax

The following changes need to be made to the Luau grammar:

```diff
- funcargs ::=  '(' [explist] ')' | tableconstructor | STRING
+ funcargs ::=  '(' [explist | namedexplist | explist ',' namedexplist] ')' | tableconstructor | STRING

+ namedexp ::= NAME '=' exp
+ namedexplist ::= {namedexp ','} namedexp
```

Note that named arguments MUST come after ordered arguments in lexical order. An error MUST be raised if this condition is not met.

### Semantics

```luau
type Foobar = (foo: number?, bar: number?) -> ()

local foobar: Foobar

foobar(bar=3.141)
```

Is equivalent to:

```luau
foobar(nil, 3.141)
```

The named argument MUST be prioritized over the respective ordered argument counterpart. The expression of an ordered argument that is ignored MUST NOT be executed. 

A lint SHOULD be used to indicate if a given ordered argument is ignored.

```luau
type Foo = (foo: number) -> ()

local foo: Foo

foo(
    2.718, -- This argument SHOULD be linted, since it is unused.
    foo=3.141
)
```

The named argument MUST be prioritized in forwarding calls. 

A lint SHOULD NOT be used here, since you could have used values in the varargs (`...`).

```luau
type Foobarqux = (foo: number, bar: number, qux: number) -> ()

local foobarqux: Foobarqux

foobarqux(
    ..., -- This SHOULD NOT be linted, since it is not unused.
    bar=1.618
)
```

In the same example, if `...` is comprised of 1, 2, and 3, the following code would be equivalent to the example given:

```luau
foobarqux(
    1,
    1.618,
    3
)
```

If named arguments use every argument in a function that MUST NOT be variadic, then forwarding calls SHOULD be linted.

```luau
foobarqux(
    ..., -- This SHOULD be linted, since it is unused.
    foo=2.718,
    bar=3.141,
    qux=1.618
)
```

If a named argument does not exist as a parameter in the function, or if the function has more than one parameter with that name, a type error MUST be raised. 

The argument MUST be ignored at runtime.

```luau
foo(
    qux=1.618 -- This MUST be a type error, as the function does not have a `qux` parameter.
)
```

```luau
type Foofoo = (foo: number, foo: number) -> ()

local foofoo: Foofoo

foofoo(
    foo=3.141 -- This MUST be a type error, as the function has two arguments with this name.
)
```

If a name argument is repeated, the duplicate SHOULD be linted, and the first argument MUST be used.

```luau
foo(
    foo=404, -- This is the value that would actually be used.
    foo=409  -- This SHOULD be linted, since it is a duplicate.
)
```

If a named argument is used in an union or intersection of functions, two conditions must be met:

1. All functions in the union/intersection MUST allow the named argument by themselves (following the same semantics described by the RFC).
2. The respective parameter for the named argument MUST have the same position across all functions in the union/intersection. 

If one of these conditions are not met, a type error MUST be raised, and the named argument is ignored.

```luau
type Correct = 
    & ( (arg: string) -> () ) 
    & ( (arg: number) -> () )

local correct: Correct

correct(arg="Hello, world!")

type Misplaced =
      ( (_: any, arg: string) -> () ) 
    & ( (arg: number, _: any) -> () )

local misplaced: Misplaced

misplaced(arg="Hello, world!") -- A type error MUST be raised here, because the argument is not in the same position across all functions

type Missing =
      ( (arg: string) -> () )
    & ( (not_arg: number) -> () )
    
local missing: Missing

missing(arg="Hello, world!") -- A type error MUST be raised here, because the argument is missing in one of the functions

type Duplicate =
      ( (arg: string) -> () )
    & ( (arg: number, arg: number) -> () )
    
local duplicate: duplicate

duplicate(arg="Hello, world!") -- A type error MUST be raised here, because the argument has a duplicate in one of the functions

```

### Implementation

This should be a compile-time feature. The compiler should decide at compile-time which register to load named arguments into. This RFC proposes that it should be based on the concrete type of the function (hence why arguments cannot be in different indices in unions/intersections).

The following function should be compiled as such:

```luau
local function foo(
    bar: (qux: number, quux: number, ...number) -> number, 
    ...: number
): number
    return bar(..., quux=42)
end

-- MOVE R1 R0
-- GETVARARGS R2 -1
-- LOADN R3 42
-- CALL R1 -1 -1
-- RETURN R1 -1
```

The named arguments being decided based on the type of the function is also quite useful to "cherry-pick" certain arguments. For example:

```luau
local function foo(bar: (...any) -> (), ...: any)
    (bar :: (any, qux: string) -> ())(..., qux="Hello, world!")
end
```

## Drawbacks

This is a feature that would require extra work on the compiler, and it isn't _strictly_ necessary, as there _are_ some alternatives to it currently, but they all have drawbacks and are generally undesirable. This feature would be a really good nice-to-have.

The proposed syntax currently works because Luau does not have assignment expressions. This feature would "reserve" the equals sign (`=`) in function call argument expressions, meaning we wouldn't be able to use it for assignment expressions later. An alternative syntax could use a colon character (`:`) instead, but this RFC opts to not use it because it is typically associated with types in Luau.

Because named arguments are resolved entirely at compile-time using static types, functions whose type cannot be determined (for example, typed as `any` or `unknown`) cannot use named arguments. This RFC proposes this as a reasonable tradeoff. It avoids any runtime performance cost and ensures that the meaning of each named argument is fully controlled by the developer.

## Alternatives

There are several common alternatives (which just shows how widespread this problem is), each with their own drawbacks:

1. Builder Pattern

   ```luau
   car.new()
      :horsepower(350.0)
      :hp(275.0)
      :owner(john_doe)
      :build()
   ```   

   - Creates an intermediate object,
   - Uses extra function calls (`N+1` for `N` fields),
   - Lacks type safety (although possible, requires type wizardry), for instance:

     ```luau
     car.new():build()
     ```

2. Table

   ```luau
   car.new { horsepower = 350.0, hp = 275.0, owner = john_doe }
   ```

   - Creates an intermediate object,
   - Prevents falling back to positional arguments, which is useful when using varargs (`...`),
   - The curly brackets are ugly boilerplate, especially in functions with mixed arguments:

     ```luau
     fs.watch("/", { recursive = true })
     ```

3. Comments

   ```luau
   example.arguments(
       nil,
       --[[ argument_a ]] true,
       nil,
       nil,
       --[[ argument_b ]] 3.14,
   )
   ```

   - Common in C++ and other languages (but unlike C++, there is no convention to do this, which means most code just doesn't),
   - A comment in Luau has a lot of unnecessary characters,
   - For optional parameters, requires manually inserting `nil` (and adding or removing them as the function signature changes),
   - If the function signature changes, this can lead to ugly silent errors:

     ```luau
     type Ugly = (argument_a: number) -> ()
  
     local ugly: Ugly
  
     ugly(--[[ argument_b ]] 0.0)
     ```

4. Strings or Enums 

   ```luau
   fs.watch("/", "recursive")
   fs.watch("/", fs.recursive)
   ```

   - Creates a string literal or uses a table lookup,
   - Can only be used for a predetermined number of options, or, if dynamic in nature, doesn't provide semantic meaning.

None of these are optimal, and named parameters don't have any of the drawbacks listed here for any of these options.

## Notes

In the future, Luau could introduce default arguments as described by [Luau RFC 91](https://github.com/luau-lang/rfcs/pull/91), and these play really well with named arguments.