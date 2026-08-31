# Support Cyclic Imports

## Summary

Now that Luau is adding classes to the language, it's much more important that we afford some way for modules to cyclically import one another.

This RFC proposes that `require()` be augmented to automatically support cyclic imports for modules that use the [`export` keyword](./export-keyword.md).  All modules participating in a cycle must use `export`; if any module in the cycle does not, the runtime will raise an error when it encounters the cycle.  This allows the runtime to close the loop and allow many cyclic import scenarios to work as desired.

## Motivation

Luau has always restricted `require()` cycles.  If the runtime encounters a cycle while evaluating a require, it raises an error and stops attempting to load the code.

Prior to our addition of classes as a builtin language feature, this was rarely a big deal because it was always possible to move functions and type definitions into different source files to break any cycles that might arise.  Luau also permits `require()` to be called within a function body.

This problem becomes much more difficult to deal with when classes are added to the mix because classes are always defined at the top level and must always be entirely defined within a single module.

Without cyclic requires, the following program cannot be evaluated.

```luau
-- folder.luau

local file = require("./file")

class Folder
    public files: {file.File}

    function add_file(self)
        table.insert(self.files, file.File {})
    end
end

-- file.luau

local folder = require("./folder")

class File
    public parent: folder.Folder
end
```

The developer is left to choose between two bad options:

1. They could introduce extra modules that just define interface types for `Folder` and `File`, or
2. Move both classes into the same script

Option 1 is laborious and sacrifices the fidelity of the type system.  Option 2 potentially means that the developer's entire program must be specified in a single script\!

## Runtime Design

For modules that use the `export` keyword, we can solve this issue by having `require` tie the knot: When it encounters a cyclic import, `require` will instead return an empty table that will later be populated with the export surface of the module.  As long as the requesting module doesn't access it at the top level during module initialization, that table will be fully populated by the time any function body actually runs.  The system will temporarily attach a metatable to surface these issues and produce a clear error message.

The restriction for cyclic module support is deliberate: 
1. `export` modules have a well-defined export surface that the runtime can reason about
2. Non-export modules behave exactly as they do today where encountering a cycle raises an error.

When a module uses `export`, the runtime automatically handles cycles without any additional boilerplate from the developer or changes to existing code.

### Runtime Algorithm

The algorithm operates in three phases: placeholder creation, short-circuit on cycle detection, and placeholder population.

`require()` will be adjusted to do the following:

#### 1. Placeholder Creation (module load start)

When a module begins executing, the runtime checks whether the module uses `export` via `lua_usesexport()`. Only if the module uses `export`:

1. An empty placeholder table is created.
2. A shared `CyclicDependencyError` metatable is attached.
3. The table is frozen via `lua_setreadonly()` to prevent any writes while the module is being evaluated.
4. The placeholder is cached under the module's cache key so that any subsequent `require()` calls for the same module will return the placeholder.

The shared metatable prevents reads and writes by raising clear error messages:

```luau
local CyclicDependencyError = {
    __index = function(self, prop)
        error(`Cannot access the exported field '{prop}' because it has a cyclic dependency on its requiring module`)
    end,
    __newindex = function(self, prop, value)
        error(`Cannot set the exported field '{prop}' because it has a cyclic dependency on its requiring module`)
    end,
    __metatable = "The metatable is locked"
}
```

#### 2. Short-Circuit on Cycle Detection (module load in progress)

When `require()` detects a cycle (i.e. the module is already on the require stack):

1. If the target module uses `export` (i.e. has a placeholder in the cache), the placeholder is noted and returned immediately to the cyclic importer.
2. If the target module does not use `export`, an error is raised at runtime, as before: `"Requested module was required recursively"`.

#### 3. Placeholder Population (module finishes executing)

When the exported module finishes executing and returns its result:

1. We check whether the placeholder was provided to any cyclic importer. If not, we simply return the result as normal.
2. If the placeholder was provided, we:
    a. Unfreeze the placeholder
    b. Copy all exported fields from the module's return value into the placeholder, and metatable (if any)
    c. Freeze the placeholder again to prevent further writes
3. The populated placeholder now replaces the module's cached result, ensuring object identity as all holders of the placeholder reference see the same fully-populated table.

### Examples

#### Basic Cyclic Import (Success)

```luau
--- folder.luau
local file = require("./file")
export local name = "folder"
export function get_file() return file.name end

--- file.luau
local folder = require("./folder")
export local name = "file"
export function get_folder() return folder.name end

--- main.luau
require("./folder")
```

Order of operations:

1. `main.luau` requires `folder.luau`. A placeholder is created for `folder` (it uses `export`) and `folder.luau` begins executing.
2. `folder.luau` requires `file.luau`. A placeholder is created for `file` (it uses `export`) and `file.luau` begins executing.
3. `file.luau` requires `folder.luau`. Cycle detected — `folder`'s locked placeholder is returned immediately.
4. `file.luau` stores the placeholder reference in local `folder` and continues executing. It defines its exports normally.
5. `file.luau` finishes. Its placeholder is populated with `{name = "file", get_folder = <function>}`.
6. `folder.luau` resumes (its require of `file` returns the now-populated placeholder). It defines its exports.
7. `folder.luau` finishes. Its placeholder is populated with `{name = "folder", get_file = <function>}`.
8. All references resolve correctly. `file.get_folder()` returns `"folder"` because by the time it's called, `folder`'s placeholder has been populated.

#### Reentrant Top-Level Access (Error)

```luau
--- folder.luau

local file = require("./file")

export class Folder
    files: {file.File}

    function add(self, f: file.File)
        table.insert(self.files, f)
    end
end

--- file.luau

local folder = require("./folder")

-- try to create a root folder at the top level
local root = folder.Folder{files={}}

export class File
    parent: folder.Folder
end

--- main.luau

require("./folder")
```

The order of operations in this program is:

1. `main.luau` requires `folder.luau`. A placeholder is created for `folder` and it begins executing.
2. `folder.luau` requires `file.luau`. A placeholder is created for `file` and it begins executing.
3. `file.luau` requires `folder.luau`. Cycle detected — `folder`'s locked placeholder is returned immediately.
4. `file.luau` attempts to access `folder.Folder`.  The placeholder is still incomplete and therefore has the `CyclicDependencyError` metatable attached to it.  We tell the developer that they `"Cannot access the exported field 'Folder' because it has a cyclic dependency on its requiring module"` and raise an exception.  The developer can use the stack trace to understand the cycle.

#### Improper Reentrant Mutation

```luau
--- folder.luau

local file = require("./file")

file.max_size = 1000

--- file.luau

local folder = require("./folder")

export const max_size = 500

--- main.luau

require("./file")
```

If we naively execute our planned resolution order, things proceed as follows:

1. `main.luau` starts evaluating `require("./file")`
2. `file.luau` starts evaluating, but is immediately blocked on `require("./folder")`
3. `folder.luau` evaluates `require("./file")`, which immediately returns with an empty table from the module cache
4. `folder.luau` inserts a property into the export table of `file`\!
5. `file.luau` resumes execution with an unexpected extra entry in its export table

`CyclicDependencyError` saves us here.  We use it to freeze the shape of `file` at step 2\.  It remains frozen until step 5\.  We therefore raise an error in step 4\.

#### Non-Export Module in Cycle (Error)

```luau
--- logger.luau
local formatter = require("./formatter")
return { log = function(msg) print(formatter.format(msg)) end }

--- formatter.luau
local logger = require("./logger")
export function format(msg) return "[INFO] " .. msg end

--- main.luau
require("./logger")
```

Order of operations:

1. `main.luau` requires `logger.luau`. No placeholder is created (`logger` doesn't use `export`).
2. `logger.luau` requires `formatter.luau`. A placeholder is created for `formatter`.
3. `formatter.luau` requires `logger.luau`. Cycle detected, but `logger` has no placeholder — error: `"Requested module was required recursively"`.

## Type System Design

The user-facing behaviour of the type inference engine should be unchanged as a result of this RFC, but the internal structure of the type checker is going to need significant changes.

Today, typechecking is driven by a class called `Frontend`.  It accepts a set of modules that need checking, builds a directed acyclic graph (DAG) from that, and checks modules one after another.

We will augment `Frontend` to instead work on all modules that can be reached from one another via `require()`. For example, given the following modules:

```
foo -> bar
bar -> baz
baz -> foo
```

`Frontend` will detect that `foo`, `bar`, and `baz` are all reachable from one another and will typecheck them together in a single pass through the solver. We call each of these sets of modules a [strongly-connected component (SCC)](https://en.wikipedia.org/wiki/Strongly_connected_component).  

Only SCCs where all members use `export` are eligible for joint typechecking. Mixed SCCs fall through to the existing typechecking behavior, which will produce an error when it encounters the cycle.

### Type System Algorithm

1. Detect strongly-connected components in the module dependency graph.
2. Created a shared type arena for all modules in the SCC, allowing types from any member to reference types from other members.
3. Pre-allocate a `BlockedType` placeholder for each module in the SCC.  This allows the typechecker to resolve references to types from other modules in the SCC, even if they haven't been fully defined yet.
4. Run constraint generation for each module in the SCC. Each module in the SCC gets its own scope and data flow graph, but all constraints flow into a single shared `ConstraintGraph`. After constraint generation for each module, the `BlockedType` placeholder is bound to the actual inferred return type.
5. Run constraint solver once over the combined constraint graph for the entire SCC. This enables cross-cycle type inference, so constraints from `foo` that depend on types from `bar`, for example, are solved together.
6. Distribute results and errors back to their original modules by their `moduleName`. Type checking runs on each module individually, and each module's public interface is then cloned and frozen via `clonePublicInterface`. The shared arena is frozen after all modules are processed.

### Large Cycle Warning

A problem that a developer might run into is that, if their application consists of a very large SCC (their whole application, perhaps\!), their incremental typechecking performance will be very bad: Luau will have to recheck all files whenever any file in the SCC has changed.

To mitigate this and put some soft pressure on the developer, we'll report a warning when we encounter an SCC that consists of too many modules.  This warning will explain that large clusters of cyclic modules can cause typechecking performance to degrade badly.  We'll allow this limit to be configured via `FrontendOptions`, defaulting to 4 modules.  The warning will list the names of the modules in the SCC, up to a configurable limit (default 10).

We need to take particular care not to break the old type solver.  We will probably need to write some extra logic to ensure that it continues to handle cyclic imports exactly as it does today.

## Drawbacks

The restrictions on how cyclic imports can be used are subtle\!  If two mutually-recursive modules need access to one another at the top level, the code will fail to load, because the placeholder has not yet been populated at that point.

For instance, the following code will fail:

```luau
--- widget.luau

local layout = require("./layout")

class Button extends layout.Container ... end
class Label ... end

--- layout.luau

local widget = require("./widget")

class Container ... end
class Row extends widget.Label ... end
```

With the described design, we will produce a sensible error, but the restriction itself is fairly complicated and is likely to confuse users.  They will likely have to think a little bit about how to adjust the design of their code and understand that cyclic references are only safe inside function bodies, not at the top level during module initialization.

## Alternatives

An earlier design passed the export table to the module as a vararg (`...`), requiring modules to opt in with `local exports = ...` or `local exports = ... or {}`. This approach had several downsides:

- It would be a breaking change for existing code that already uses the vararg mechanism for other purposes.
- It required boilerplate in every module that wanted cycle support.
- It was ambiguous whether a module intended to participate in cycles or was simply using varargs for another reason, making it harder for the runtime to reason about the module's export surface.
- It allowed modules to attach metatables to their export table, creating edge cases that could be difficult to handle correctly.
