# Require by String with Aliases

**Status**: Implemented

## Summary

We need to add intuitive alias and paths functionality to facilitate the grouping together of related Luau files into libraries and allow for future package managers to be developed and integrated easily.

## Motivation

The Roblox engine does not currently support require-by-string. One motivation for this RFC is to consolidate require syntax and functionality between the Roblox engine and the broader Luau language itself.

Luau itself currently supports a basic require-by-string syntax that allows for requiring Luau modules by relative or absolute path. Unfortunately, the current implementation does not support aliases.

### Package management

While package management itself is outside of the scope of this RFC, we want to make it possible for a package management solution to be developed in a way that integrates well with our require syntax.

To require a Luau module under the current implementation, we must require it either by relative or absolute path:

```luau
-- Requiring Roact by absolute path
local Roact = require("C:/LuauModules/Roact-v1.4.2")
```

If we wanted to change the location of this package on disk, we would have to modify all files that require the old path above, which essentially boils down to manual package management. This approach is tedious and prone to mistakes.

To solve this, we introduce path and alias configuration in this RFC, which would allow for future package management software to maintain package paths and versions by directly modifying configuration files.

This would also create simple and readable require statements for developers.

```luau
-- Suppose "@src" is an alias for the same directory as "../../../../../"

-- Instead of this:
local result = require("../../../../../HelperModules/graphing")

-- We could have this:
local result = require("@src/HelperModules/graphing")
```

## Design

### Aliases

Aliases can be used to bind an absolute or relative path to a convenient, case-insensitive name that can be required directly.

```json
"aliases": {
    "Roact": "C:/LuauModules/Roact-v1.4.2"
}
```

Based on the alias map above, you would be able to require Roact directly with an `@` prefix:

```luau
local Roact = require("@Roact")
```

Or even a sub-module:

```luau
local createElement = require("@Roact/createElement")
```

Aliases are overrides. Whenever the first component of a path exactly matches a pre-defined alias, it will be replaced before the path is resolved to a file. Alias names are also restricted to the charset `[A-Za-z0-9.\-_]`. We restrict the charset and make them case insensitive because we envision alias names to be primarily used as package names, which tend to be case insensitive and alphanumeric. They also must be preceded by an `@` symbol.

#### Versioning

Aliases are simple bindings and aren't concerned with versioning. The intention is to allow package management software to leverage aliases by automatically adding and updating the alias map to reflect a package's dependencies. For manual versioning, a user could always "version" their aliases: `@MyModule-v1`, `@MyModule-v2`, etc.

#### Library root alias

In the past, it has been proposed to define an alias (e.g. "`@`") to represent the root directory of a file's encapsulating library.
- However, the concept of a "Luau library" and its root directory is not yet rigorously defined in Luau, in terms of folder/file structure.
- In the future, we may add a `package.json` file or something similar that marks the root directory of a library, but this is outside of the scope of this RFC, which primarily focuses on improving require-by-string.
- For the time being, this functionality will remain unimplemented for this reason. The alias "`@`" will remain reserved for now, meaning it cannot be overridden.

Of course, users can still use the alias map to explicitly define this behavior with a named alias:

```json
"aliases": {
    "src": "../"
}
```

#### Current limitations of aliases

- Alias names cannot contain the directory separators `/` and `\`.
- Aliases can only occur at the beginning of a path.

#### Configuring alias maps: .luaurc

As part of this proposal, alias maps will be configured in `.luaurc`, which follows a JSON-like syntax.

Proposed structure of an alias map:
```json
{
    "aliases": {
        "alias1": "/path/of/alias1",
        "alias2": "/path/of/alias2"
    }
}
```

Missing aliases in `.luaurc` are inherited from the alias maps of any parent directories, and fields can be overridden.

If an alias is bound to a relative path, the path will be evaluated relative to the `.luaurc` file in which the alias was defined.
If an alias resolves to an aliased path, the aliased path is resolved by checking the current `.luaurc` file and falling back to parent directories if needed.
This search continues iteratively if a chain of aliases must be resolved; if a cycle is detected, alias resolution will fail with an error.

Finally, providing support for alias maps within the Roblox engine is out of the scope of this RFC but is being considered internally.

### Paths

**This section is now obsoleted by [this RFC](https://github.com/luau-lang/rfcs/pull/56):**
The `paths` array described below is no longer supported.

Similar to [paths in TypeScript](https://www.typescriptlang.org/tsconfig#paths), we will introduce a `paths` array that can be configured in `.luaurc` files. Whenever a path is passed to `require` and does not begin with `./` or `../`, the path will first be resolved relative to the requiring file. If this fails, we will attempt to resolve paths relative to each path in the `paths` array.

The `paths` array can contain absolute paths, and relative paths are resolved relative to `.luaurc` file in which they appear.

#### Example Definition

With the given `paths` definition (`.luaurc` file located in `/Users/johndoe/Projects/MyProject/src`):
```json
"paths": [
    "../dependencies",
    "/Users/johndoe/MyLuauLibraries",
    "/Users/johndoe/MyOtherLuauLibraries",
]
```

If `/Users/johndoe/Projects/MyProject/src/init.luau` contained the following code:
```luau
local graphing = require("graphing")
```
We would search the following directories, in order:
- `/Users/johndoe/Projects/MyProject/src`
- `/Users/johndoe/Projects/MyProject/dependencies`
- `/Users/johndoe/MyLuauLibraries`
- `/Users/johndoe/MyOtherLuauLibraries`

### Symlinks

Symlinks carry some security concerns; for example, a link's target might exist outside of the project folder in which the link was defined. For the first version of this implementation, symlinks will not be supported and will be treated as ordinary files.

If we do implement symlinks in the future, we will likely use our own limit to the number of symlinks to "follow through" for cross-platform compatibility. It is also possible that we will add a new configurable property in `.luaurc` that will allow developers to toggle whether or not to resolve symlinks.

Similar examples:
- https://www.typescriptlang.org/tsconfig#preserveSymlinks
- https://webpack.js.org/configuration/resolve/#resolvesymlinks

## Use Cases

### Alias map

Using alias maps, Luau developers can require globally installed Luau libraries in their code without needing to specify their locations in Luau scripts.

```
luau-aliases-project
├── .luaurc
└── src
    └── module.luau
```

For example, if we wanted to require `Roact` in `module.luau`, we could add the following alias to `.luaurc`:

```json
{
    "aliases": {
        "Roact": "/Users/johndoe/LuauLibraries/Roact/src"
    }
}
```

Then, we could simply write the following in `module.luau`, and everything would work as intended:
```luau
local Roact = require("@Roact")
local Component = require("@Roact/Component")
```

If we ever wanted to change the version of `Roact` used by `luau-aliases-project`, we would simply change the absolute path to `Roact` in `.luaurc`. By abstracting away the exact location of globally installed libraries like this, we get clean, readable code, and we make it easier for a future package manager to update dependencies by modifying `.luaurc` files.

### Paths array

**This section is now obsoleted by [this RFC](https://github.com/luau-lang/rfcs/pull/56):**
The `paths` array described below is no longer supported.

The `paths` configuration variable provides convenience and allows Luau developers to build complex, well-organized libraries. Imagine the following project structure:

```
luau-paths-project
├── .luaurc
├── dependencies
│   └── dependency.luau
└── src
    └── module.luau
```

If `.luaurc` contained the following `paths` array:
```json
{
    "paths": ["./dependencies"]
}
```

Then, `module.luau` could simply require `dependency.luau` like this:
```luau
local dependency = require("dependency")

-- Instead of: require("../dependencies/dependency")
```

Using the `paths` array allows Luau developers to organize their projects however they like without compromising code readability.

### Large-scale projects in Luau

For large-scale Luau projects, we might imagine that every dependency of the project is a Luau project itself. We might use an organizational structure like this to create a clean hierarchy:

```
large-luau-project
├── .luaurc
├── subproject-1
├── subproject-2
└── subproject-3
```

We can provide the following alias in `large-luau-project/.luaurc`:

```json
{
    "aliases": {
        "com.roblox.luau": "."
    }
}
```

This way, each subproject directory can contain its own source code, dependencies, and `.luaurc` configuration files, while also inheriting the `com.roblox.luau` alias from `large-luau-project/.luaurc`.

This allows us to refer to other subprojects like this, regardless of the exact location of the requiring file in `large-luau-project`:
```luau
local subproject1 = require("@com.roblox.luau/subproject-1")
```
### Roblox Specifics
In the Roblox engine, a similar aliasing system could be implemented. Assuming a central package management system were available, a Roblox Script could contain local Roact = require("@Roact"), and everything would "just work".

## Drawbacks
## Backwards Compatibility
This alias system introduces a new layer to require that wasn't previously there. However, the advantages of this system are expected to outweigh the complexities it introduces.

## Alternatives

### Different ways of defining aliases

#### Defining paths/aliases directly in the requiring file

Rather than defining paths/alias maps in an external configuration file, we could alternatively define paths/aliases directly in the files that require them. For example, this could manifest itself through an extension of the `--!` comment syntax or introduce new syntax like `--@<ALIAS> = @<PATH>`.
```luau
--@"Roact" = @"C:/LuauModules/Roact-v1.4.2"
local Roact = require("@Roact")

-- Same as this:
local Roact = require("C:/LuauModules/Roact-v1.4.2")
```

Some potential issues with this approach:
- Could lead to Luau file headers becoming cluttered.
- Would probably lead to substantial copy-and-pasting between modules in the same library, as they would likely need to share certain paths/aliases.
- Using configuration files for paths/alias maps allows modules to share aliases while still providing the flexibility to override if needed. This approach does not support inheritance and overriding in an obvious way.
- Removes the layer of abstraction that is provided by external paths/alias maps. This might also blur the scope of package managers. The software would need to directly modify lines of code in `.luau` files, rather than modifying configuration files.

#### Defining configuration files in a non-JSON format

Alias maps could alternatively be defined in specially named `.luau` files themselves, or at least adhering to Luau syntax. This approach is somewhat unappealing, however, as it would require package management software to parse Luau syntax. JSON syntax is well-understood and well-supported, which would likely facilitate the development of package management software.
