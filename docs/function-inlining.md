# No support for user inlining (yet)

## Summary
<!-- https://luau-lang.org/performance#function-inlining-and-loop-unrolling -->
Luau's bytecode compiler is smart about inlining functions. We would rather focus on improving this automation than risk the chance of its misuse.

## Motivation

**context**: function inlining replaces function calls with the function's definition to reduce overhead costs.

Whether the goal is to improve instruction locality or perform context specific optimizations, <!-- force the compiler to optimize the function body in specific contexts --> function inlining can be a great resource for developers. At `-02` or higher, the compiler is able to automatically decide if functions ought to be inlined. In the currrent state of the language, exposing this to users directly could do more harm than good. Not only do these use cases only cater to a tiny subset of the language's users, it exposes our bytecode compiler to FILL when the larger subset who may not have a working understanding of the compiler adopts it. For Luau, this would happen at around the 5000th recursive call.

```luau
local function sum(lst: {number})
  return sumAcc(0, 1, lst)
end

@inline
local function sumAcc(acc: number, i: number, lst: {number})
  return if i > #lst then acc else sumAcc(acc + lst[i], i + 1, lst)
end

local s = sum({1, 2, 3})
```

Consider this "inlined" recursive call, it attempts to sum up numbers in a list but it never terminates as `sum` is recursively called on `lst` which does not shrink. If Luau attempted to inline such a call, it would unroll this and one or two things could happen. If we did have an unroll limit then we would attempt our unrolling up until that point and end up making the program larger than it needs to be or thrashing our stack. The stack will grow beyond the system's stack limit and could potentially eat up resources meant for other processes.
<!-- one or two also sounds funny, change -->

More specifically, due to limited number of cases where the compiler supports inlining, exposing this feature would not be entirely useful. This section further explores common cases were `@inline` would not apply.

1. Exported a `table`
```luau
-- A.luau
local mod = {}

@inline
function mod.foo(t)
  print(t.foo)
end

return mod -- no table.freeze

-- B.luau
local A = require("./A")

local mod = {}

function mod.foo()
  A.foo(mod)
end

return table.freeze(mod)
```

Consider this example which exports a mutable table `mod`. Module `B` imports this table and calls it within another function, but because the compiler decides what implementation of this method should be called at runtime through the metatables, it makes the VM unable to guarantee that `A.foo()` has not been mutated somewhere between the boundaries of both modules making `mod.foo` not suitable for an inline. Even with a frozen table being exported by module `B`, its function foo could be mutated before its return leading us back to the same issue.

<!-- the compiler looks around and tries to decides what implementation of this method should be called. -->
<!-- table freeze locks the table -->
<!-- making it a somewhat okay case to apply inlining too except cases where -->
<!-- function properties of the table could be "mutated?" by the importing module -->
<!-- why would exporting a "warm?" table not be suitable for inlining? -->
<!-- the compiler decides what function will be called at runtime through the metatable so if another module imports mod and overrides foo -->
<!-- like virtual functions, dynamic dispatch? -->
<!-- With Luau tables, the VM performs some voodoo magic based to try to quickly discover the implementation of this method through the metatable. -->


2. "OOP" code
```luau
local Account = {}
Account.__index = Account

function Account.new(name)
  return setmetatable({ name = name, balance = 0 }, Account)
end

@inline
function Account:deposit(bal)
  self.balance += bal
end
```

Even OOP style code, this mutable resolution problem persists. In this code sample, we define a base `Account` class with a deposit method. However, if an instance of this class were made, say `DavesAccount`, it could go on to overload the deposit method making this another case that is not suitable for inlining.


3. Metamethods
```luau
local Vec3 = {}
Vec3.__index = Vec3

function Vec3.new(x, y, z)
  return setmetatable({ x = x, y = y, z = z }, Vec3)
end

@inline
function Vec3:__add(rhs)
  return Vec3.new(self.x + rhs.x, self.y + rhs.y, self.z + rhs.z)
end
```
In this example, we declare `Vec3` with an inlined `__add` (`+`) operator overload. To make this inlining work, we would need to do one of two things, (1) confirm the `rhs` and `lhs` share the same shape (i.e, they both have`x`, `y` and `z`) or (2) ensure the `rhs` has a similar operator overload. Due to dynamic dispatch, we cannot make either guarantee making this case unsuitable for overloads.

## Design
None (yet).

## Drawbacks
None (yet).

## Alternatives
Automatic function inlining in Luau is very efficient. We suggest opening github issues with code samples of legitimate cases that we can use to further improve Luau's automatic (function inlining and unrolling) optimizations.

