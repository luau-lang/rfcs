# Abstract module paths and `init.luau`

**Status**: Implemented

## Summary

We will redefine `foo/init.luau` to be the contents of the module named foo for
the purpose of resolving relative `require()` calls within its body.

We will do this because a significant portion of our users today rely on the
idiom that `init.luau` represents the module's contents, especially when using
Rojo to build for Roblox.

## Motivation

First, consider an example filesystem:

```
./foo.luau
./package
./package/init.luau
./package/foo.luau
./package/dependency.luau
```

Luau maps this filesystem into a module tree.  Module paths largely correspond
to filesystem paths, but they are distinct ideas.  Given the filesystem above,
Luau considers the following module paths to be valid:

```
foo
package
package/init
package/foo
package/dependency
```

When Luau is running on a conventional filesystem, we presently offer some
special behaviour to afford the ability for modules to both contain other
modules, and for them to themselves export values and types: The module path
`package` is treated as an alias to `package/init`.

Library authors can use this feature to afford a package hierarchy with a root
package that exports all of the most important, basic functionality, plus
submodules that either contain implementation details or additional public APIs.

This feature works well, but it's incompatible with the idiom that Rojo, a
popular tool in our ecosystem today, employs when building artifacts for Roblox.

This incompatibility stems from how requires are resolved within the body of
`init.luau`.

Because we consider `package/init` to be an ordinary Luau module, it is
considered to be rooted at `package`.  Therefore, a relative
`require('./foo')` call will resolve to `./package/foo.luau` rather than
`./foo.luau`.

Rojo, by contrast, considers `./package/init.luau` to belong to the containing
directory.  A developer using Rojo must instead write
`require('./package/foo')`.  This is only necessary when the script is named
`init.luau`.

Because of this incompatibility, scripts with require-by-string uses in Rojo
projects will not work outside of Roblox places and vice versa. Since
cross-runtime compatibility is an important goal of the require-by-string
project, we would like to rectify this.

## Design

First, we introduce a new abstraction: a _module path_.  Module paths refer
either to _modules_ or _directories_.

For the purposes of navigation, both modules and directories are functionally
identical: modules and directories can both have children, which could
themselves be modules or directories, and both types can have at most one
parent, which could also be either a module or a directory.

The thing that separates a module from a directory is precisely that modules
represent source code that can be imported via the `require()` function.
Directories, by contrast, are merely organizational units.

The central feature of this RFC is about how module paths correspond to
filesystem paths: A module path refers to a module if it corresponds either to a
`.luau` file or to a directory that contains a file named `init.luau`.  A module
path refers to a directory if it refers to a filesystem directory that lacks an
`init.luau` file.

More concretely, Luau will no longer consider relative requires from a package
`init.luau` file to resolve relative to the script itself.  It will instead
resolve relative to the location of the abstract module it represents, i.e. the
location of its parent folder in a filesystem context.

Secondly, we recognize an unfortunate side effect of this change: code within
`package/init.luau` is forced to write `require('./package/dependency')` when it
specifically wants to carry out the ordinary task of importing a subordinate
module.

We propose to alleviate this with a special import alias `@self` that resolves
to the path to the current module.

```lua
-- package/init.luau

local foo = require('./foo')     -- This pulls in the outer foo!
local foo = require('@self/foo') -- import package/foo
```

## Drawbacks

On a filesystem, Rojo's behaviour is frankly very weird!  A reader must know the
name of the current source file in order to know what `require('./foo')` will
do, and if a developer renames a file to or from `init.luau`, they will be
forced to rewrite every `require()`.

We could someday mitigate this with a new lint: If a module is the parent of other
modules, it is poor style for it to directly import sibling or parent modules.
Files named `init.luau` should never issue a require that resolves to a sibling
module.  In our example, we would warn if `package/init.luau` were to
`require('./foo')` because that reaches outside of its folder.

Further, as require-by-string has been live on Roblox for a little while, there
is some risk that changing things now will break existing code.  Roblox will use
live telemetry to assess the impact of this change before we move forward.

## Alternatives

### Patching Rojo

Adjusting Rojo to match what Luau does today without breaking any existing
application code is tricky and most likely requires something far weirder than
the change outlined in this RFC.

Crucially, most code written using Rojo predates string-based requires.  All of
this code must continue to run exactly as-is.

This puts us in quite a bind:

* If we map `foo/init.luau` to an actual `ModuleScript` at `foo/init` in the
  Roblox data model, then instance-based requires break.  This rules out clever
      solutions like aliasing modules and compatibility shims.
* If we don't, then we need some other magic Roblox rule that changes the
  require resolution rules
    1. Only for scripts that, on the Rojo side, are named `init.luau`, and
    2. Only for string-based requires
