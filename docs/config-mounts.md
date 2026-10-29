# Support mounting modules via configuration

## Summary

Add a `mounts` option to Luau configuration files (`.luaurc` and `.config.luau`): a flat map from a name to a path that places that module or directory into the abstract module hierarchy as a child of the directory containing the config. Module identity follows the mounted (abstract) path, not the location of the source on disk. This lets tools (or users) layer configuration above user code without editing user-owned files, and lets the same source on disk appear in several places with distinct runtime identities and dependency aliases.

## Motivation

Build systems, package managers, bundlers and workspace tools need to tell the runtime about project structure that they generate. Discussion on [#265](https://github.com/luau-lang/rfcs/pull/265) identified two needs that today's configuration can't meet.

**1. Tool configuration above user configuration.** A tool should be able to provide aliases and settings that apply to user code (e.g. `@Packages`) without editing the user's `.config.luau`. Hierarchical configuration almost supports this already: a tool-owned config in an ancestor directory is inherited by user code, and the user's nearer config takes precedence. What's missing is a portable way for a tool-owned directory to sit *above* user code without owning the project root. OS symlinks can do this, but they aren't reliably portable across platforms.

**2. On-disk deduplication with distinct identities.** Package managers want to store one copy of a package on disk (e.g. in a content-addressed store) and reuse it across many places in a project or across projects. Each place it appears must keep its own runtime identity (its own require-cache entry) and its own outgoing dependency aliases.

For example, `MySQL` may be compatible with several versions of `OpenSSL`, and dependency resolution may pick different versions in different places. The shared copy of `MySQL` must stay immutable, so the edge from a given `MySQL` instance to its `OpenSSL` must be defined by the project that uses it, not stored in `MySQL`'s directory.

Aliasing the same directory from several places doesn't meet this need: the copies share one require-cache identity and don't inherit configuration from where they're referenced.

## Design

### Syntax

A configuration file may define `mounts`, a map from a name to a path:

```luau
-- added to the existing Config type
mounts: { [string]: string }?
```

In `.luaurc` (JSON), `mounts` is a top-level key. In `.config.luau`, it lives under `luau`, like `aliases`.

- **Name (the key):** a single path component, which becomes a child of the directory containing the config file. Names must be non-empty, must not be `.` or `..`, must not contain `/`, and must not begin with `@`.
- **Path (the value):** a relative path (beginning with `./` or `../`) to the mounted module or directory, resolved as described in [Mount paths](#mount-paths). It follows the same module resolution rules as require-by-string, so a mount may target a module file (without its extension) or a directory. Aliases are not permitted. Runtimes may additionally accept absolute paths (e.g. for a global package store); this is implementation-defined.

### Semantics

**Navigation.** When the require navigator moves from directory `D` to a child named `n`, it checks the `mounts` declared by the config file at `D`. If `n` is mounted, navigation continues into the mounted source. Mounts are **not inherited**: a mount declared at `D` only creates a child of `D`.

**Identity.** A module's identity (require-cache key and chunkname) is its abstract path, so the same source mounted at two locations gives two independent modules. Mounting doesn't hide the original: a source that is also reachable at its own location has a separate identity there. In the first example below, `src/` is reachable as both `src` and `build/src`, and loading it both ways in one program creates two instances.

**Ancestry.** Configuration lookup follows the abstract path. For a module inside a mount, its ancestors are the configs above and at the directory declaring the mount, followed by any configs inside the mounted source between its root and the module. Configs that are ancestors of the source *on disk* (e.g. a `.luaurc` in a package store) are not applied. As today, nearer configuration overrides farther configuration.

**Relative navigation.** `..` from a mount point moves to the abstract parent (the directory declaring the mount), not the source's parent on disk, so relative requires and alias paths inside a mount stay within the abstract hierarchy. `@self` likewise resolves to the module's abstract location.

**Collisions.** If a mount's name matches an existing module or directory in the same directory (on disk or another mount), an error is raised. Names are compared as module names, so a mount named `foo` collides with `foo.luau`, `foo/init.luau`, or a `foo/` directory. This follows the existing rule that `.luaurc` and `.config.luau` in the same directory is an error.

#### Mount paths

Every config file has two locations:

- its **abstract location**: where it appears in the abstract module hierarchy, and
- its **source location**: where it is actually loaded from (on disk, or whatever address the runtime uses to load a node's source).

Outside a mount, the two are the same. Inside a mount they differ: a config at source location `.store/MySQL-8.0.0/` might have abstract location `build/Packages/MySQL-1/MySQL/`.

**Mount paths** resolve against the config's **source location**, because a mount describes where code comes from. **Alias values and requires** resolve against the **abstract location**, because they describe edges within the hierarchy.

A mount path therefore always refers to a real (unmounted) location and cannot traverse other mounts; to place another mount's contents somewhere, mount its source directly. This keeps mounted subtrees self-contained: a tool's output, or a package, means the same thing wherever it is mounted.

#### Nested mounts

Mounts declared by configs inside a mounted source are honoured, so mounts chain: if `A` mounts `B`, and `B`'s config mounts `C`, then `C` appears beneath `A/B`.

#### Cycles

While navigating, if a mount resolves to the source location of a directory already on the current abstract path, an error is raised. Because mount paths always refer to real locations, this catches indirect cycles (e.g. `A` mounts `B`, which mounts `A`'s source) as well as direct ones.

### Examples

#### Tool configuration above user code

```
project/
├── build/
│   └── .luaurc            -- tool-owned
└── src/
    ├── .config.luau       -- user-owned: @client, @server, @shared
    └── ...
```

`build/.luaurc`:

```json
{
    "aliases": { "Packages": "./Packages" },
    "mounts": { "src": "../src" }
}
```

When running `build/src/main.luau`, user code inherits `@Packages` from the tool's config. The user's `src/.config.luau` is nearer, so its settings win in the event of a conflict.

#### Deduplicated packages with per-instance dependencies

Per-instance configuration comes from a tool-owned **wrapper directory**, whose config mounts the package as a child and defines that instance's aliases. The package sits one level below the wrapper because its own root config can't share a directory with the wrapper's.

`ServiceA` and `ServiceB` both depend on `MySQL`, but resolution picks `OpenSSL` 3.1 for one and 1.1 for the other. The store holds one copy of `MySQL`:

```
project/
├── .store/                          -- content-addressed, never modified
│   ├── ServiceA-1.0.0/
│   ├── ServiceB-2.0.0/
│   ├── MySQL-8.0.0/
│   ├── OpenSSL-3.1.0/
│   └── OpenSSL-1.1.1/
├── build/                           -- tool-owned
│   ├── .luaurc
│   └── Packages/
│       ├── .luaurc
│       ├── ServiceA/.luaurc
│       ├── ServiceB/.luaurc
│       ├── MySQL-1/.luaurc
│       ├── MySQL-2/.luaurc
│       ├── OpenSSL-3/.luaurc
│       └── OpenSSL-1/.luaurc
└── src/
```

`build/.luaurc`:

```json
{
    "aliases": {
        "ServiceA": "./Packages/ServiceA/ServiceA",
        "ServiceB": "./Packages/ServiceB/ServiceB"
    },
    "mounts": { "src": "../src" }
}
```

`build/Packages/.luaurc`:

```json
{ "lint": { "*": false } }
```

`build/Packages/ServiceA/.luaurc`:

```json
{
    "aliases": { "MySQL": "../MySQL-1/MySQL" },
    "mounts": { "ServiceA": "../../../.store/ServiceA-1.0.0" }
}
```

`build/Packages/ServiceB/.luaurc`:

```json
{
    "aliases": { "MySQL": "../MySQL-2/MySQL" },
    "mounts": { "ServiceB": "../../../.store/ServiceB-2.0.0" }
}
```

`build/Packages/MySQL-1/.luaurc`:

```json
{
    "aliases": { "OpenSSL": "../OpenSSL-3/OpenSSL" },
    "mounts": { "MySQL": "../../../.store/MySQL-8.0.0" }
}
```

`build/Packages/MySQL-2/.luaurc` (same source as `MySQL-1`):

```json
{
    "aliases": { "OpenSSL": "../OpenSSL-1/OpenSSL" },
    "mounts": { "MySQL": "../../../.store/MySQL-8.0.0" }
}
```

`build/Packages/OpenSSL-3/.luaurc`:

```json
{ "mounts": { "OpenSSL": "../../../.store/OpenSSL-3.1.0" } }
```

`build/Packages/OpenSSL-1/.luaurc`:

```json
{ "mounts": { "OpenSSL": "../../../.store/OpenSSL-1.1.1" } }
```

- `Packages/MySQL-1/MySQL` and `Packages/MySQL-2/MySQL` share one source on disk but are separate modules. Code inside each finds `@OpenSSL` in its own wrapper's config and resolves it to a different version.
- The store is never modified; all dependency edges live in the wrapper configs.
- Lints are off beneath `Packages` only, so user code in `src` is unaffected.

#### Nested mounts across tools

Tool `A` mounts the output of tool `B`, which itself mounts a vendored dependency:

```
project/
├── a-out/
│   └── .luaurc       { "mounts": { "b": "../b-out" } }
├── b-out/
│   ├── .luaurc       { "mounts": { "c": "../vendor/c" } }
│   └── ...
└── vendor/c/
```

When navigating `a-out/b/c`, the mount `c` is declared by `b-out/.luaurc`. That config's abstract location is `a-out/b`, but its source location is `project/b-out`, so `../vendor/c` resolves to `project/vendor/c`, as tool `B` intended. Had mount paths resolved against the abstract location, it would resolve to `a-out/vendor/c`, and tool `B`'s output would break as soon as another tool mounted it.

### Composition between tools

Tools compose by nesting rather than merging: each tool owns a directory and mounts the next layer (another tool's output, or user code) beneath it. Precedence follows the existing nearest-wins rule, so composition requires no extra build step.

### Runtime requirements

- When navigating to a child, check the config at the current location for `mounts`.
- Track both the abstract location and the source location of each node.
- Key the require cache and chunknames by abstract path.
- Resolve config ancestry, `..`, requires and alias values by abstract location, and mount paths by source location.
- Raise errors for collisions and cycles.

Runtimes without a filesystem can implement mounts wherever they already attach configuration to nodes in their module hierarchy, using whatever address they load a node's source from as its source location. Runtimes may restrict mount targets, for example to locations within a project root.

### Unresolved questions

- **Package config vs. wrapper edges:** a mounted package's own config is nearer than its wrapper's, so it can shadow the dependency aliases the wrapper provides. Should dependency aliases be reserved for wrapper configs, or is convention (package authors don't declare external dependencies) enough?
- **Alias inheritance into packages:** aliases are inherited by everything beneath the config that defines them, including mounted packages. In the deduplication example, `@ServiceA` and `@ServiceB` are visible inside `MySQL`. Should a config be able to stop inheritance at a given point, such as a wrapper?
- **Absolute paths:** should absolute mount paths be standardised, or left implementation-defined?
- **Editor context:** how should analysis tools choose which abstract location to use for a source file that is mounted in several places?

## Drawbacks

- **More work during navigation:** the navigator must check for `mounts` at each downward step. Runtimes already load these configs to resolve aliases, but it is still new logic and possible overhead.
- **Two kinds of relative path:** within one config, mount paths resolve against the source location while alias values resolve against the abstract location. Outside mounts the two coincide, but inside a mount authors need to know which rule applies.
- **Entry point inside the mounting hierarchy:** code must be run and analysed from the tool's directory (e.g. `build/`). Running `src/` directly won't see tool-provided configuration unless tooling handles it.
- **Many small files, no single manifest:** per-instance configuration needs one wrapper directory and config file per package instance. The runtime reads more files at startup, and a project's resolved structure is spread across many configs rather than described in one place.
- **An extra path segment:** mounted packages sit one level below their wrapper, which appears in require paths and chunknames.
- **Configuration can shape the tree:** in `.config.luau`, `mounts` can be computed by arbitrary Luau. Tool-owned configs can use `.luaurc` to stay declarative, but user-written configs may not.
- **Same-location conflicts are errors:** tools that want to configure the same location must nest instead, which may be awkward for some workflows.
- **Analysis tools need context:** type checkers and language servers must implement mounts too. A source file in a shared store can appear at several abstract locations with different aliases, so each instance is checked separately, diagnostics may differ between instances, and their exported types are distinct.
- **Abstract chunknames:** stack traces and error messages show abstract paths (e.g. `build/Packages/MySQL-1/MySQL/...`), not the file on disk, so tooling may need to map them back to source locations.
- **Adoption:** in Luau's reference implementation, the `.luaurc` parser rejects unknown keys (`Unknown key mounts/...`), while the `.config.luau` parser silently ignores unknown keys under `luau`. Runtimes without support will therefore fail to load a `.luaurc` that uses `mounts`, but will silently drop `mounts` from a `.config.luau`, which surfaces later as unresolved requires.

## Alternatives

**A separate project-wide file ([#265](https://github.com/luau-lang/rfcs/pull/265), `.modules`).** This describes the hierarchy and its sources in one machine-generated file, and covers the same cases (a `.modules` `Submodule` corresponds to a wrapper directory with a mount). A single manifest has real advantages: the runtime reads one file at startup, the whole structure can be reviewed in one place, and it fits runtimes without a filesystem naturally. As currently drafted, though, it adds a new file type, leaves the merge rules for composing files unspecified, produces a derived file that can go stale, and nests machine configuration beneath user configuration so that machine settings take precedence. Mounts instead reuse the configuration files and precedence rules Luau already has.

**Mounts with attached configuration.** Mount entries could take `{ source, config }`, with `config` able to declare further mounts. That would let a whole project be described in one file without wrapper directories, but it makes mounts recursive, adds a third level of precedence between the mounting directory and the package, and duplicates what a config in a wrapper directory already does. It could be added later if wrapper directories prove insufficient.

**OS symlinks.** These give the layering in the first example, but they aren't reliably portable across platforms or runtimes.

**Letting `.config.luau` require generated sibling files.** This lets tool-provided aliases be merged into user configs, but it requires users to edit each config and doesn't help with deduplication or identity.

**Doing nothing.** Tools would keep using runtime-specific mechanisms (e.g. reading lockfiles directly), and deduplication with distinct identities would remain impossible.

## Prior Art

- **[pnpm](https://pnpm.io/symlinked-node-modules-structure)** hard links each package from a content-addressed store into its own instance directory (`node_modules/.pnpm/foo@1.0.0/node_modules/foo`), and symlinks that instance's dependencies alongside it in the same `node_modules` folder. Wrapper directories follow the same layout, including placing the package one level below its instance directory. Because Node resolves symlinks to their real paths, pnpm must hard link a package again for each distinct set of dependencies it resolves to (which happens with peer dependencies); mounts give each instance its own identity from a single copy on disk, without OS links.
- **[Import maps](https://html.spec.whatwg.org/multipage/webappapis.html#import-maps)** take the opposite approach: a page-wide manifest (multiple maps on a page are merged into one), closer in shape to `.modules`, that remaps specifiers globally and per scope. Scopes are similar to the aliases a wrapper provides to the instance it mounts, but modules are identified by resolved URL, so mapping two specifiers to the same URL yields one instance rather than two.
