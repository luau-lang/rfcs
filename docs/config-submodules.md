# Machine-generated module hierarchy configuration

## Summary

Introduce `.modules`: a machine-writable sibling to `.config` that configures a whole abstract module hierarchy from a single, easily-manipulated file.

- Tools can generate and compose these in one place, without touching user-owned configuration.
- Runtimes read a single composed `.modules` and can mostly pass it to Luau.
- Luau adopts minimal resolution logic that runtimes can choose to extend, if they care to.

## Motivation

Build systems, workspaces, project managers, bundlers, package installers and other tools need some way of communicating machine-generated configuration to the runtime, in a way that doesn't clobber user intent.

We don't want to hardcode special language features for any one tool. Every language feature has maintenance cost and runtime weight, so this proposal aims for something generically useful, which applies to use cases beyond those of any single tool. 

In particular, we aim to build something generically useful across all Luau runtimes, not something specifically fitted to Roblox's needs or Lute's shape. 

We consulted closely and carefully with the Luau team to land on a shape for this proposal. We believe what we arrived at here is both powerful enough to meet the bar for any Luau build tooling, but also simplified and restricted enough to be readily adoptable by runtimes of any shape, adopting minimal logic.

### What do machines need to configure?

**Luau configuration**

Aliases are the obvious example: build systems, workspaces, and package installers all want to provide locations for things.

That said, the scope of configuration should be larger than "just aliases" and should likely cover all the things you'd put in a `.config` today. For example, you may want to disable type checking / lints on an `outputs/` directory, or specify a global that should be assumed present for a project.

**Deduplication**

A module that appears multiple times in a project may deduplicate to a single `.luau` file on-disk. This is commonly done by build systems, bundlers and package managers to reduce disk footprint and in-memory usage.

On-disk deduplication must have no impact on runtime behaviour, meaning disk-deduplicated modules still have their own runtime identities and unique require-cache keys. This is a correctness invariant, not a preference: a package that is resolved against different dependencies in different places must not be merged into one instance just because its source bytes happen to be identical.

(Internally: see our Confluence space for the lpm "robust deduplication" investigation that uncovered these invariants, which is not linked here.)

### Why can't machines write to `.config`?

**Configuration is resolve-only**

When we established `.config.luau` as a Luau module that *resolves to* static configuration ([RFC](https://rfcs.luau.org/config-luauconfig.html)), we made it unsafe for a machine to assume it could *edit* the on-disk contents of `.config` files.

While most config files will be straightforward table literals that are safe to edit, *that may not be true in all cases* - the user may have applied more complex logic like variables, merging, or procedural generation which makes certain members - or the whole table - unmodifiable without reasoning through a program flow.

Furthermore, `.config` stores *user intent*, which heightens the risk from "clobbering another machine's definitions" to "clobbering the user's own preferences".

While it's possible for tooling to play nicely and behave, it takes incredible ecosystem-wide discipline, and one misbehaving project can cause damage in the best case, or lock the ecosystem into its mechanics in the worst case of a misbehaving pivotal dependency.

**Deduplication-by-aliasing breaks things**

`.config` supports aliasing the same file hierarchy from multiple locations, which looks like deduplication, but it violates the invariant above.

In particular:

- There's no way to merge them on-disk without merging them in the require cache.
  - This prohibits any kind of "instancing" or global stores - one module on disk is one identity at runtime, which forces duplication mechanically.
- The location in the abstract module hierarchy is not preserved.
  - They are not considered "children of" their original parents once deduplicated.
  - They don't receive configuration rules they'd otherwise have inherited.
  - Require navigation cannot escape the deduplicated hierarchy.

### Consequences

Machines can only reliably generate - or regenerate - `.luau` files they own.

- They should be separate from `.config` to preserve user intent.
- They should be separate from other machines' files to prevent clobbering or having to reason through other machines' program flow.

The files they all emit should, *at some point,* be something the runtime can consume.

- The runtime should consume the *composition* of all of these files.
- The user should have agency over how these files compose; what takes precedence, what gets adopted, what gets ignored.

We need something beyond deduplication-by-alias.

- The abstract module hierarchy and its configuration should not be strictly tied to on-disk layout.
- Source code on-disk is treated as a source of bytes, not as a literal part of the abstract module hierarchy.

## Design

We introduce `.modules` as a sibling to `.config`, which encodes data about a whole module hierarchy at this location.

Unlike `.config` - which continues to represent user intent - `.modules` is designed to be machine-readable and composable by tooling.

### Workflow

Any agent that wants to contribute configuration writes a `.modules`-compatible module that it owns:

```
my-game/
├── .config              -- user-owned
├── my-game.modules      -- user-owned
├── lpm.modules          -- machine-generated
├── darklua.modules      -- machine-generated
└── src/
```

A tool takes in an ordered list of module names, and produces a composed `.modules` where the runtime expects it. For example, this could be a Lute subcommand (hypothetical syntax):

```
$ lute use-modules my-game lpm darklua
```

The runtime only needs the final `.modules`, which encodes the whole composed configuration as one bundle:

```
my-game/
├── .modules             -- read by the runtime
└── src/
```

The runtime reads one composed `.modules`, which is the the only part this RFC standardises. Composition beyond that is a tooling concern - if you disagree with these tools, you're able to write your own, by design.

Tooling is free to enhance this experience as it sees fit: for instance, `lute run` can regenerate `.modules` when its inputs change, tracking them in a git-ignorable cache. That way, users who don't care about `.modules` never need to see it. 

### File format

`.modules` defines an abstract module hierarchy to be "mounted to" this location in the overall abstract module hierarchy. Separately, it maintains a table mapping requirable modules to locations where that module's source code may be found.

Refer to the "Worked example" appendix below to see examples of a populated `.modules` file.

```luau
type DotModules = {
    submodules: { [string]: Submodule },
    sources: { [number]: SourceSet }
}

type Submodule = {
    config: DotConfig?,
    sourceIndex: number?,
    submodules: { [string]: Submodule }?
}

type SourceSet = { Source }
type Source =
    | { type: "luau/module", navPath: string }
    | { type: string, [unknown]: unknown }
```

The submodule hierarchy intentionally only contains abstract structure and standardised configuration. It maps directly to the abstract module hierarchy that Luau runtimes already support ([RFC](https://rfcs.luau.org/abstract-module-paths-and-init-dot-luau.html)), and should not be extended by tooling to store other kinds of metadata.

The source map is where runtimes and tooling can look for metadata *about* a module in a way that composes - in particular, information about how a runtime should locate a module's source code.

The file must return a Luau table of the following `DotModules` type, where `DotConfig` is the `Config` type returned by `.config.luau`:

### Submodules

Each entry in `submodules` is a node in the abstract module hierarchy, mounted beneath the location of the `.modules` file.

- A submodule with a `sourceIndex` is a module, and can be required. Its source code is found via the corresponding entry in `sources`.
- A submodule without a `sourceIndex` is a directory - require navigation can't terminate there, but it can still carry configuration, just like an on-disk folder.
- `config` is optional, just as it is optional on-disk. Where present, it applies to that submodule and its descendants in the same way as a `.config` at that location would.
- Everything declared in `.modules` is within scope of the `.config` at the location of the `.modules` file.
- Where a submodule's name collides with something on-disk at the same location, `.modules` wins.

### Sources

File paths and module paths look alike, but are interpreted very differently, so they live in different tables. Submodules refer to sources by index.

Many submodules can share one source. They deduplicate on-disk, but keep their own identities at runtime, because identity is determined by position in the abstract module hierarchy rather than by source location. Tools can therefore deduplicate their files without knowing which runtime will execute them.

Like a clipboard holding the same content in several formats, each source set lists interchangeable sources. The runtime uses the first `type` it understands, and ignores the rest. Beyond that, the only required field is `type`.

For now, we only standardise one source type: `luau/module`, which locates source code via the standard require navigator. Its `navPath` is resolved relative to the location of the `.modules` file. Since these are locations that Luau by definition can already resolve, any runtime with a require navigator understands `luau/module`, making it the portable baseline. Any `.config` or `.modules` found at the target location is interpreted as being atop the hierarchy it was mounted into.

Runtimes may add non-standard source types without breaking the universal standard path. For example, Lute could support filesystem paths (in particular, absolute paths) without breaking standard Luau's portability, Roblox packages could share one copy of a dependency, and `lute compile` could emit a `.modules` whose only sources are bundled bytecode.

### Composition

All of this is designed to compose neatly: a tool that wishes to combine two `.modules` can do so by evaluating each `.modules`'s return value, performing a table merge, and writing the new `.modules` out to disk.

## Worked example

Consider a game project using lpm which depends on `Jest` and `JestGlobals` from [jest-roblox](https://github.com/Roblox/jest-roblox) 3.20.1. Those two packages resolve to 51 packages in total, which lpm installs into a content-addressed folder:

```
my-game/
├── .modules   -- 🔎 read by the runtime
├── src/
│   └── … user's source code …
└── .lpm/
    ├── Jest-3.20.1-jbmv90iz/
    ├── JestGlobals-3.20.1-4m5snyo8/
    ├── JestTypes-3.20.1-5s5j1j31/
    ├── RegExp-0.3.0-7i8lla56/
    └── … 47 more …
```

The composed `.modules` could look like this:

```luau
-- my-game/.modules
return {
    -- Define the abstract module hierarchy the require navigator will explore here.
    -- Where there's a name collision, `.modules` wins over what's on disk,
    -- and everything here is within scope of the `.config` from this module.
    submodules = {
        ["Packages"] = {
            -- Any submodule without a sourceIndex is treated as a directory;
            -- require navigation can't terminate here, but configuration
            -- can still be specified, just like an on-disk folder.
            config = { luau = { lint = { ["*"] = false } } },

            submodules = {
                -- These modules all have sourceIndex, so they are requirable.
                -- The runtime checks the sources table to find the code (see below).
                ["Jest"] = {
                    sourceIndex = 1,
                    config = { luau = { aliases = { JestCore = "../JestCore" } } }
                },
                ["JestGlobals"] = {
                    sourceIndex = 2,
                    config = { luau = { aliases = { Expect = "../Expect", JestEnvironment = "../JestEnvironment", JestTypes = "../JestTypes", LuauPolyfill = "../LuauPolyfill" } } }
                },
                ["JestTypes"] = {
                    sourceIndex = 3,
                    config = { luau = { aliases = { RegExp = "../RegExp" } } }
                },
                -- config is optional, just as it is optional on-disk.
                ["RegExp"] = {
                    sourceIndex = 4
                },
                -- ...and 47 more packages of the same shape
            },
        },
    },
    -- Each set lists interchangeable sources; a runtime uses the first type it understands
    sources = {
        -- Use the standardised require navigator to locate source code files.
        -- luau/module acts like you symlinked the target in (but in a more cross-platform way),
        -- interpreting any `.config` or `.modules` it finds as being atop *your* hierarchy.
        [1] = { { type = "luau/module", navPath = "./.lpm/Jest-3.20.1-jbmv90iz" } },
        [2] = { { type = "luau/module", navPath = "./.lpm/JestGlobals-3.20.1-4m5snyo8" } },
        [3] = { { type = "luau/module", navPath = "./.lpm/JestTypes-3.20.1-5s5j1j31" } },

        -- Anyone can add their own source types if they'd like.
        -- They can store custom user metadata, be processed and transpiled by tooling,
        -- or even be understood directly by runtimes if they so choose.
        [4] = {
            { type = "lute/file", path = "~/.lpm/global/RegExp-0.3.0-7i8lla56" },
            { type = "rbx/core-packages", id = "RegExp-0.3.0" },
            { type = "luau/module", navPath = "./.lpm/RegExp-0.3.0-7i8lla56" }
        },
        -- ...and 47 more
    },
}
```

## Drawbacks

**Another configuration surface**: Users and tools now have two places where configuration can live. Someone debugging an unexpected alias or lint setting may need to check both `.config` and `.modules`, and understand how they interact. Our original design pushed back on this, but the Luau team pointed out that this is relatively unavoidable if you're aiming to keep user intent separate from machine-generated configuration.

**Runtime adoption**: Every runtime that wants to benefit must implement reading `.modules`, including Roblox. Until a runtime does, tools targeting it can't rely on `.modules`. We seriously weighed the impact of this during discussion, and it motivated our drive to keep runtime cost minimal, as it's been rightfully pointed out in previous RFCs that users outside the Roblox and Lute ecosystems already don't fully implement our spec; a complex feature would drive a further wedge here.

**Staleness**: The composed `.modules` is derived from its inputs. If it isn't regenerated after an input changes, the runtime will see out-of-date configuration. Tooling such as `lute run` can mitigate this, but runtimes that don't regenerate it themselves leave this to the user.

## Alternatives

### **Let machines edit** `.config`

Rejected for the reasons in the motivation - `.config` is resolve-only and represents user intent.

### **Generated alias files imported by** `.config`

Our team's original draft proposed a shape like this, but the Luau team convinced us it was not a good idea, and explicitly redirected us away from proposing this solution.

Tools could write a generated table beside each `.config`, which `.config` then requires and adopts as it sees fit. This needs `.config.luau` to gain the ability to require a sibling file, which it can't today. It also means composing a generated file in every folder a tool touches, rather than one file per tool, and it doesn't separate module identity from source location, so it doesn't solve deduplication. 

### **Imperative resolution**

Rather than declaring a hierarchy, the runtime could call into user code which does all the pathing, similar to how type functions allow arbitrary user code to determine types. 

This is more powerful, but has more pitfalls: it allows "non-euclidean" relative paths, side effects, and termination concerns similar to those of type functions. It also risks building something that only makes sense in one runtime or on a filesystem.

In the words of the Luau team:

> "if the file’s purpose is to be machine-generated, it should probably not be Turing complete".

Since `.modules` is intended to be machine-generated, a declarative format is a better fit.

### **A non-Luau format**

 `.modules` could use JSON, TOML or similar. Both we, and the Luau team, are okay with this.

This doesn't necessarily change composability: a Luau file can be resolved into a returned table, just as a JSON file can be. We would likely round-trip to a Luau table format anyway if writing Lute tooling to merge tables, and tooling that writes `.config` files today already "speaks Luau".

We use Luau for consistency with `.config.luau`, so embedders don't need to support another format, and because we don't yet have any other strong impetus to change it.

### **Do nothing**:

Each tool continues to communicate configuration in its own way - for example, a runtime reading a lockfile directly to infer a hierarchy (the package-aware run we already built for the `lute pkg run` prototype and do not want to standardise).

This ties runtimes to specific tools, and leaves deduplication without a solution that preserves module identity. It's also bad stewardship on our part: we'd be introducing something for ourselves and not solving anyone else's use case. The need for *some* solution here is foregone as deduplication is physically impossible without a new runtime feature, so - even if it isn't exactly this proposal - we should be doing *something* more than forcing everyone into a shape we assembled for our own tools only.

---

## Prior Art

The `SourceSet` format is designed to mimic clipboard and drag-and-drop APIs, which hold the same content in several formats (MIME types) and let the consumer pick the one it understands. Source sets apply the same idea to module sources, so that runtimes can pick the one it understands. Universal standardised formats can live alongside bespoke runtime formats to let runtimes optimise for their needs without loss of portability.

[Import maps](https://github.com/WICG/import-maps) let a web page remap module specifiers globally and per-scope, separately from where the module source lives - similar to per-submodule aliases in `.modules`.

Node.js [conditional exports](https://nodejs.org/api/packages.html#conditional-exports) let a package list several targets for the same export, with the runtime picking the first condition it matches.

Within Luau, `.modules` builds directly on [.luaurc](https://rfcs.luau.org/config-luaurc.html), [.config.luau](https://rfcs.luau.org/config-luauconfig.html), require by string with aliases, and abstract module paths. It reuses the existing Config type rather than introducing new configuration options, and its modules and directories have the same meaning as in the abstract module paths RFC.

