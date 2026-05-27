# Support Cyclic Imports

## Summary

Now that Luau is adding classes to the language, it's much more important that we afford some way modules to cyclically import one another.

This RFC proposes that `require()` be augmented to pass an export table into the module.  This allows the runtime to close the loop and allow many cyclic import scenarios to work as desired.

## Motivation

Luau has always restricted `require()` cycles.  If the runtime encounters a cycle while evaluating a require, it raises an error and stops attempting to load the code.

Prior to our addition of classes as a builtin language feature, this was rarely a big deal because it was always possible to move functions and type definitions into different source files to break any cycles that might arise.  Luau also permits `require()` to be called within a function body.

This problem becomes much more difficult to deal with when classes are added to the mix because classes are always defined at the top level and must always be entirely defined within a single module.

Without cyclic requires, the following program cannot be evaluated.

```luau
-- A.luau

local B = require("./B")

class A
    public children: {B.B}

    function add_child(self)
        table.insert(self.children, B.B {})
    end
end

-- B.luau

local A = require("./A")

class B
    public parent: A.A
end
```

The developer is left to choose between two bad options:

1. They could introduce extra modules that just define interface types for `A` and `B`, or
2. Move both classes into the same script

Option 1 is laborious and sacrifices the fidelity of the type system.  Option 2 potentially means that the developer's entire program must be specified in a single script\!

## Runtime Design

For modules that return tables, we can solve this issue by having `require` tie the knot: When it encounters a cyclic import, `require` will instead return an empty table that will later be populated with the export surface of the module.  As long as the requesting module doesn't access it at the topmost global scope, that table will eventually be populated and everything will work out.  The system will temporarily attach a metatable to surface these issues and produce a clear error message.

There are subtle edge cases to consider here:

1. If a module fails to access a property from another module because of a cycle, Luau needs to clearly communicate what happened.
2. If a module acquires a reference to an incomplete module due to a cycle, it should not be able to mutate that module\!
3. Today, many modules return something other than a table.  It is okay if these modules do not support participation in cycles, but they still need to work as-written.
4. This proposal requires modules to be adjusted to work with cyclic imports.  Modules that have not been adjusted need to work exactly as-written.
5. When a non cycle-supporting module appears in a cycle, Luau still needs to communicate the problem to developers clearly.

### Algorithm

`require()` will be adjusted to do the following:

1. First, save the current export table's metatable away, and replace it with a new `CyclicDependencyError` metatable.  This metatable prohibits reads and writes to the table by raising an exception with a clear error message.
2. Look up the requested module in the cache to see if it has already been loaded or begun loading
3. If a module is already present in the cache, return it immediately. Otherwise,
    a. Populate the cache with a fresh table.
    b. Pass this new table to the target script as its sole argument and evaluate it.  This table can be accessed within the script via `...` at the top level.
    c. Once the module has been evaluated and returned a value, test to see if that value is the same as the table that was passed in.  If they are not the same, set `CyclicDependencyError` as the metatable on the original export table. (the one that wound up not being used)  The table will also be frozen for good measure.
    d. Replace the module cache result with the result of the module
4. Restore the current export table's metatable.

This approach handles many cases, but has an important limitation:  A module that participates in a cycle can freely access imported symbols within function bodies, but not at the top level.  This is because those imported symbols cannot be guaranteed to have been evaluated yet.

Step 3c covers an important edge case: In this design, the `require` function sometimes speculatively returns a table with the expectation that it will eventually become the export surface of the requested module.  If it is not, then we have a problem: We have already provided that table to other requesting modules\!  Luckily, this can only happen when we encounter a cycle between modules that do not accept the export table, so all we need to do is to mark that speculative export table as something that cannot be used.

In almost all reasonable cases, we expect the current module's export table to have no metatable.  We specify that steps 1 and 4 save and restore it just to handle the odd case where someone is adding a metatable to the exports.  We do not consider this to be good style at all, but this adjustment is very easy.

The new metatable `CyclicDependencyError` can roughly be defined as follows:

```luau
local CyclicDependencyError = {
    __index = function(self, prop)
        error(`Cannot access the exported field {prop} because it has a cyclic dependency on its requiring module`)
    end,
    __newindex = function(self, prop, value)
        error(`Cannot set the exported field {prop} because it has a cyclic dependency on its requiring module`)
    end,
    __metatable = "The metatable is locked"
}
```

In the absence of `export`, a script must be updated to support cyclic requires by making a small edit: Instead of creating an export table directly with `{}`, the script should accept it from `...` like so:

```luau
local exports = ...

function exports.foo() end
exports.MY_CONSTANT = true

return exports
```

If necessary, the script could instead adopt a compatibility shim so that it works in older Luau environments that do not implement this RFC: `local exports = ... or {}`

The new `export` keyword will be updated to handle this automatically.

This algorithm satisfies a bunch of important properties:

While existing code will not support cyclic `require()` calls, it will continue to work as-written.  Modules that return non-table values will also continue to work exactly as expected.

If necessary, a module could be crafted to work with or without support for cycles by instead starting with `local exports = ... or {}`.

### Examples

#### Reentrant Accesses

```luau
--- A.luau

local B = require("B")

export class Tree
    children: {B.Node}

    function append(self, prototype: B.Node)
        -- In this example, we suppose that the tree needs to
        -- insert a clone of the passed argument.
        table.insert(self.children, B.Node(prototype))
    end
end

--- B.luau

local A = require("A")

-- create a global tree for some reason
local t = A.Tree{children={}}

export class Node

end

-- main.luau

require("A")
```

The order of operations in this program is:

1. `main.luau` starts importing `A.luau`
2. `A.luau` starts importing `B.luau`
3. `B.luau` attempts to import `A.luau`.  We sense the cycle and short circuit; the incomplete module `A` is returned immediately.
4. `B.luau` attempts to access `A.Tree`.  The value `A` is still incomplete and therefore has the `CyclicDependencyError` metatable attached to it.  We tell the developer that a cyclic dependency error has been encountered and raise an exception.  The developer can use the stack trace to understand the cycle.

#### Improper Reentrant Mutation

```luau
--- A.luau

local B = require("B")

B.foo = "bar"

--- B.luau

local A = require("A")

export const foo = "foo"

--- main.luau

require("B")
```

If we naively execute our planned resolution order, things proceed as follows:

1. `main.luau` starts evaluating `require("B")`
2. `B.luau` starts evaluating, but is immediately blocked on `require("A")`
3. `A.luau` evaluates `require("B")`, which immediately returns with an empty table from the module cache
4. `A.luau` inserts a property into the export table of `B`\!
5. `B.luau` resumes execution with an unexpected extra entry in its export table

`CyclicDependencyError` saves us here.  We use it to freeze the shape of `B` at step 2\.  It remains frozen until step 5\.  We therefore raise an error in step 4\.

## Type System Design

The user-facing behaviour of the type inference engine should be unchanged as a result of this RFC, but the internal structure of the type checker is going to need significant changes.

Today, typechecking is driven by a class called `Frontend`.  It accepts a set of modules that need checking, builds a DAG from that, and checks modules one after another.

We will augment this class to instead work on one strongly-connected component\* at a time.  All modules within an SCC use the same arena and are typechecked together in a single pass through the solver.

\* A "strongly connected component" is a set of 2 or more modules that all mutually `require()` one another. (eg if you had a require chain of `A -> B -> C -> A`, the SCC would consist of `A`, `B`, and `C`)

A problem that a developer might run into is that, if their application consists of a very large SCC (their whole application, perhaps\!), their incremental typechecking performance will be very bad: Luau will have to recheck all files whenever any file in the SCC has changed.

To mitigate this and put some soft pressure on the developer, we'll report a warning when we encounter an SCC that consists of too many modules.  This warning will explain that large clusters of cyclic modules can cause typechecking performance to degrade badly.  We'll allow this limit to be configured via `FrontendOptions`.

We need to take particular care not to break the old type solver.  We will probably need to write some extra logic to ensure that it continues to handle cyclic imports exactly as it does today.

## Drawbacks

The restrictions on how cyclic imports can be used are subtle\!  If two mutually-recursive modules need access to one another at the top level, the code will fail to load.

For instance, the following code will fail:

```luau
--- A.luau

local B = require("B")

class ClassAOne extends B.ClassOne ... end
class ClassATwo ... end

--- B.luau

local A = require("A")

class ClassBOne ... end
class ClassBTwo extends A.ClassATwo ... end
```

With the described design, we will produce a sensible error, but the restriction itself is fairly complicated and is likely to confuse users.  They will likely have to think a little bit about how to adjust the design of their code.

## Alternatives

This RFC goes to some lengths to specify how cycle support works for modules that don't use the new `export` keyword.  An alternative design would be to, instead of using `...` to hold the export table, to put it in some other place that's inaccessable within the current module as it's being evaluated.  This would simplify some of the edge cases because there would be no way, for instance, to attach a metatable to the current module's exports.

The current proposal is not to do this because `...` is a preexisting mechanism that works really well to solve this class of problem and because the edge cases don't seem very difficult to deal with.
