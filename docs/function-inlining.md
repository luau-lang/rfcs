# No support for user inlining (yet)

## Summary
<!-- https://luau-lang.org/performance#function-inlining-and-loop-unrolling -->
Luau's bytecode compiler is smart about inlining functions. We would rather focus on improving this automation than risk the chance of its misuse.

## Motivation

**context**: function inlining replaces function calls with the function's implementation to reduce overhead costs.

Whether the goal is to improve instruction locality or perform context specific optimizations, <!-- force the compiler to optimize the function body in specific contexts --> function inlining can be a great resource for developers. At `-02` or higher, the compiler is able to automatically decide if functions ought to be inlined. In the currrent state of the language, exposing this to users directly could do more harm than good. Not only do these use cases only cater to a tiny subset of the language's users, it exposes our bytecode compiler to stack overflow errors when the larger subset who may not have a working understanding of the compiler adopts it. For Luau, this would happen at around the 5000th recursive call.

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

Consider this "inlined" recursive call, it attempts to sum up numbers in a list but it never terminates as `sum` is recursively called on `lst` which does not shrink. If Luau attempted to inline such a call, it would unroll this up until the 5000th recursive call and end up thrashing our stack. The stack will essentially grow beyond the system's stack limit and could potentially eat up resources meant for other processes.

More specifically, due to limited number of cases where the compiler can even consider inlining, exposing this feature would not be entirely useful. This section further explores common developer patterns that may adopt `@inline` in ways the compiler cannot handle.

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

This example exports a mutable table `mod` in module `A`. Module `B` imports this table and calls it within another function, but because the compiler tries to decide what implementation of this method should be called at runtime through the metatables, it is unable to guarantee that `A.foo()` has not been mutated somewhere between the boundaries of both modules making `mod.foo` not suitable for an inline. Even with a frozen table being exported by module `B`, its function foo could also be mutated before its return leading us back to the same issue.

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

With Object Oriented Programming style code, this mutable resolution problem persists. In this code sample, we define a base `Account` class with a deposit method. However, if an instance of this object were made, say `DavesAccount`, it could go on to overload the deposit method making this another case not suitable for inlining.


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
In this example, we declare `Vec3` with an inlined `__add` (`+`) operator overload. To make this inlining work, we would need to do one of two things, (1) confirm the `rhs` and `lhs` share the same shape (i.e, they both have`x`, `y` and `z` keys) or (2) ensure the `rhs` has the same operator overload, giving the compiler additional confidence. However, due to dynamic dispatch, we cannot make either guarantee making this case unsuitable for inlines.

## Drawbacks
The major drawback here is that this prevents developers the opportunity to fine-tune performance-critical optimizations, forcing them to rely on the compiler for fine-grained optimizations, even if their use case may call for it. Doing this assures us that developers have no control over raising the maximum cost of Luau programs, especially in the case where developers would chose to riddle their code base with inlines to make the code "faster".

## Alternatives
Automatic function inlining in Luau is already very efficient. We suggest opening Github issues with code samples of legitimate cases that we can use to further improve the compiler's automatic (function inlining and unrolling) optimizations. [File a Github issue](https://github.com/luau-lang/luau/issues/new?assignees=&labels=bug&projects=&template=bug_report.md&title=Inlining%20Support)



