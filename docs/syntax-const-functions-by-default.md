# Make `function name()` const by default

## Summary

Make `function foo() ... end` equivalent to `const function foo() ... end`. A function statement with a single identifier introduces an immutable local binding instead of assigning to an existing variable or global.

## Motivation

Functions are typically defined once and used throughout their scope, rather than reassigned.

Currently, `function foo()` performs an assignment rather than introducing a new binding. Depending on the surrounding scope, it may overwrite an existing local variable or assign to a global. This behavior makes function declarations dependent on existing bindings and can lead to unintended reassignment.

Making `function foo()` equivalent to `const function foo()` ensures that it always introduces an immutable local binding, prevents accidental reassignment, and avoids implicit global assignments.

The shorter syntax also makes immutable function declarations the default, while `local function` and explicit assignments remain available when reassignment is needed.

## Design

A function statement of the form `function NAME` introduces a new local binding with the same semantics as `const function NAME`.

The name is in scope within its own body, allowing recursive calls. The declaration follows normal lexical scoping rules and can shadow an existing binding, even within the same scope. Reassignment to the new binding is an error, including from nested functions.

```luau
function factorial(n: number): number
    if n <= 1 then
        return 1
    end
    return n * factorial(n - 1)
end

assert(factorial(4) == 24)

factorial = function() return 0 end -- error
```

Use `local function` to declare a reassignable local function, or `foo = function() ... end` to assign a function to an existing variable or global.

This rule applies only to a single identifier. The forms `function T.foo() ... end` and `function T:foo() ... end` retain their existing behavior of assigning to table fields.

Function expressions are unaffected and inherit their binding semantics from the declaration or assignment that stores them.

## Drawbacks

This is a backwards-incompatible semantic change. Currently, a bare function statement assigns to an existing local if the name resolves to one, or to the global environment otherwise. Under this proposal, it always introduces a new local binding.

Code that relies on assignments to existing locals, implicit global definitions, or forward declarations may therefore change behavior.

For example, the following program produces a different result without necessarily producing an error:

```luau
local foo = function() return 1 end

local function read()
    return foo()
end

function foo() return 2 end

print(read()) -- 2 today, 1 under this proposal
```

Here, `read` continues to capture the original `foo`, while references after the new declaration resolve to the new binding.

Forward declarations are similarly affected:

```luau
local foo

function foo()
    return 42
end
```

Under this proposal, the function declaration shadows the original local rather than initializing it. Code that relies on the original binding must use an explicit assignment instead.

This also affects the forward-declaration pattern described in the Export by Value RFC:

```luau
export local f, g

function f()
    return g()
end

function g()
    return 42
end
```

Under the existing export proposal, these function statements initialize the previously declared exported bindings. With the new semantics, they instead introduce separate local bindings, leaving the original exports uninitialized.

The equivalent code would require explicit assignments:

```luau
export local f, g

f = function()
    return g()
end

g = function()
    return 42
end
```

Code that attempts to reassign a function declared with the new semantics would also be rejected.

The prevalence of these compatibility issues in existing Luau code is an open question and may require further investigation before adoption.

## Alternatives

Do nothing. Developers can already use `const function` to explicitly declare immutable local functions. This preserves backwards compatibility but retains assignment semantics for bare function statements.

## Prior Art

`export function foo() ... end` introduces an immutable local binding and exports its value. It establishes a precedent within Luau for function declarations that implicitly use `const` without requiring the keyword. This proposal extends that behavior to bare function declarations. [Export by Value RFC](https://rfcs.luau.org/export-keyword.html)