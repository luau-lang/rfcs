# Mounts

## Summary

`.mounts.json` is a standard file format which describes navigation behaviour for require that can be implemented by an embedder outside of Luau's abstract API. This RFC focuses on two use cases:

- instantiate an isolated module hierarchy (a *mount*) based on (potentially deduplicated) directories on disk, and
- define the aliases that take you from one module hierarchy to another, in a way that keeps require navigation stateless.

The purpose of this ecosystem RFC is to align interested implementors towards a standard format that can be understood universally between runtimes, developer tools, and LSP servers.

## Motivation

`.mounts.json` separates two things that Luau's filesystem conventions currently tie together: where a module's source/bytecode is stored, and where the loaded module sits in the abstract module hierarchy. Identity (require-cache key and chunkname) follows position in the hierarchy, so that instantiations of the same source/bytecode can be configured differently and cached independently. This allows for source/bytecode deduplication, even in the presence of libraries which depend on the behaviour of the require cache and having a unique referential identity per-instantiation.

The obvious "simple" alternative is to not do this; allow source code to be duplicated on disk, and you can just copy around instances of libraries on-disk to their position in the abstract module hierarchy. However, it's important to note this is actually a non-functional solution: it only works for a subclass of libraries designed to be stateless and unaffected by duplicate references. Examples of common patterns that violate this assumption are:

- table equality checks (by reference), used often in OOP paradigms to check class type
- the Symbol pattern
- locals defined directly in modules
- side effects caused by executing a module chunk
- tables, functions or userdata used as table keys

These patterns are so widely adopted - in popular community libraries not authored by us, even - that we believe this direction to be a non-starter. Even in a world where disk space is unlimited and memory budget is infinite, you would still want deduplication for managing module identity, for the same reason the require cache is necessary: tons of Luau code just can't handle being duplicated at runtime.

## Design

### Scope

`.mounts.json` is not read by Luau itself; Luau's require API is intentionally stateless and abstract. Luau has no awareness of filesystems or other kinds of hierarchy directly. Instead, embedders can choose to read the file and apply it to their model of the abstract module hierarchy, whatever form that takes.

Since `.mounts.json` is a project-level organisational tool, it imposes certain assumptions about how Luau modules are organised, which may not be compatible with runtimes that leverage Luau's abstract require navigation in ways that don't look like traditional filesystem navigation. For that reason, implementing `.mounts.json` isn't a prerequisite for embedding Luau.

### Mounts

`.mounts.json` declares hierarchies to be added to the abstract module hierarchy, called *mounts*. Mounts are a separate concept from aliases: each mount is given a string key in `.mounts.json`, contains enough information to locate the files and configure what is visible to it, and a single mount can be navigated into by any number of aliases via that mount's string key. The string key doesn't escape `.mounts.json`.

Every mount is isolated: you cannot navigate out of that location, and outside configuration doesn't leak in. Relative navigation such as `../` works anywhere inside a mount, but stops at the mount's root, regardless of where an alias landed inside it. The aliases seen while you're in a mount are controlled by `.mounts.json`, which underwrites what the mount's local `.config` specifies.

This allows "islands" of Luau code to be declared in one well-known place, configured externally where it's possible to define their relationships, and then used by the current codebase without restructuring.

### Aliases

Aliases in `.mounts.json` refer to a mount using `@` followed by the mount's key, and can include a path after the key, just like `.config` aliases can today. For example, `"MySQL": "@MySQL-1/src"` navigates into the `MySQL-1` mount, then to `src` inside it.

`.mounts.json` defines aliases for two distinct consumers. Firstly, it lets you define aliases on the current module (and inheritors), so that it can enter into mounts from the outside. Secondly, it lets you define aliases for individual mounts, so mounts can navigate to other mounts without internal awareness of each other.

`.config` can refer to aliases defined by `.mounts.json`, the same way that `.config` can refer to pre-defined aliases today. However, mount keys stay internal to `.mounts.json` for the sole purpose of referring to unique mounts, so they are not accessible in `.config`.

### Locating `.mounts.json`

`.mounts.json` discovery works the same way as `.config` discovery. Luau's concerns end at "modules have configuration defined on them abstractly"; how a module carries a `.mounts.json` is up to the embedder, just as it is for `.config`.

Only the nearest `.mounts.json` applies; multiple `.mounts.json` files are not merged.

`.mounts.json` files inside of mounts are ignored; only the "base" module hierarchy may define a `.mounts.json` configuration.

### Interaction with `.config`

`.mounts.json` underwrites `.config.luau` when both are present on the same module. Hierarchically, `.mounts.json` is considered configuration on a module, so config discovery treats it as terminating: when searching upwards for an alias, the search stops at the first `.mounts.json` it finds. That is to say, a `.config.luau` above a `.mounts.json` loses to the `.mounts.json` and is overruled.

This doesn't limit navigation: Luau code outside of a mount can still navigate above a `.mounts.json` with `../` as normal.

Embedders can implement this using the existing `to_alias_fallback` mechanism in `Luau.Require`. During alias lookup, the embedder reports any configuration above the nearest `.mounts.json` as absent. The upward search then only sees `.config.luau` files up to and including the `.mounts.json` node, and if none of them define the alias, `to_alias_fallback` resolves it from `.mounts.json`.

### Identity

How module identity (require-cache key and chunkname) is represented is left to the embedder. The only requirements are that distinct mounts get distinct cache keys, and that a mount reached by several aliases gets one, so that instantiations of the same source/bytecode are cached independently while every route into a single mount shares it.

### Sources

Each mount's `source` field tells the runtime where to read the module from. Since this varies across embedders (some read from disk, some pull from a central registry, some pull from the internet, etc) we type this field as `unknown`. The point of `source` is to standardise where implementors store and look for this information. We cannot standardise how to understand and interpret that information across an unbounded and unknown number of implementors.

Tools support the embedders they care about; for example, luau-lsp understands Lune and Lute's conventions for locating files on-disk, so it can support those.

### File format

`.mounts.json` is a JSON file. This is done to ensure the file contains no control flow or Turing complete logic, and so can be machine edited in parts, rather than needing to be overwritten from scratch. The choice of format is still up in the air; see Alternatives.

The file has the following shape, written as a Luau type:

```luau
type MountsFile = {
    mounts: { [string]: Mount }?,
    aliases: { [string]: string }?,
}

type Mount = {
    source: unknown,
    aliases: { [string]: string }?,
}
```

Mounts carry aliases only. Any other configuration for code inside a mount comes from that mount's own `.config`.

For example, two services depend on the same version of `MySQL`, but dependency resolution picks a different version of `OpenSSL` for each. Both `MySQL` mounts share one source, but are separate modules with their own aliases:

```json
{
    "mounts": {
        "ServiceA": { "source": "./.store/ServiceA-1.0.0", "aliases": { "MySQL": "@MySQL-1/src" } },
        "ServiceB": { "source": "./.store/ServiceB-2.0.0", "aliases": { "SQL": "@MySQL-2/src" } },
        "MySQL-1": { "source": "./.store/MySQL-8.0.0", "aliases": { "OpenSSL": "@OpenSSL-3/lib" } },
        "MySQL-2": { "source": "./.store/MySQL-8.0.0", "aliases": { "OpenSSL": "@OpenSSL-1/lib" } },
        "OpenSSL-3": { "source": "./.store/OpenSSL-3.1.0" },
        "OpenSSL-1": { "source": "./.store/OpenSSL-1.1.1" }
    },
    "aliases": {
        "ServiceA": "@ServiceA/src",
        "ServiceB": "@ServiceB/src",
        "Database": "@MySQL-1/src"
    }
}
```

Here, `source` is a path because that's what this example embedder understands; other embedders may use other values.

Some requires, and where they lead:

| Required from | Require | Result |
|---|---|---|
| user code | `require("@ServiceA")` | `src` in `ServiceA` |
| user code | `require("@Database")` | `src` in `MySQL-1`; the same module `ServiceA` gets from `@MySQL` |
| `ServiceB` | `require("@SQL")` | `src` in `MySQL-2`; a separate module from `MySQL-1`, despite sharing its source |
| `MySQL-2` | `require("@OpenSSL/hash")` | `lib/hash` in `OpenSSL-1` |
| `MySQL-2/src` | `require("../assets/certs")` | `assets/certs` in `MySQL-2`; `../` works up to the mount's root |
| `MySQL-1` | `require("@ServiceA")` | error; top-level aliases aren't visible inside mounts |

## Drawbacks

### Another configuration surface

Aliases can now come from two places: `.config.luau` files, and the nearest `.mounts.json`. Someone debugging an unexpected alias may need to check both, and understand that `.mounts.json` underwrites any `.config.luau` on the same module while cutting off the search for `.config.luau` files above it. We think this is a reasonable price for keeping configuration about mounts outside of the locations it configures (see Alternatives), but it is one more thing for users and tool authors to learn.

### Silently ignored files

Since only the base module hierarchy may define a `.mounts.json`, any `.mounts.json` found inside a mount is ignored without an error. We decided not to error because it's not well understood what such an error would look like or manifest as.

A potential failure mode is a package that was developed as its own project, with its own `.mounts.json` for its dependencies, then published with that file still in place. Once the package is mounted elsewhere, its `.mounts.json` doesn't apply; the consuming project's `.mounts.json` decides what the package can see. That is usually what you want, but it may surprise someone who expected the package's own file to be honoured.

Supporting nested `.mounts.json` was considered. However, a nested file would declare mounts per *instance* of the mount it sits in, so mount identities would have to be chained through every level of nesting, which we consider too difficult for embedders to maintain.

### Adoption

Code relying on `.mounts.json` won't run on runtimes that don't implement it. Since implementing `.mounts.json` isn't a prerequisite for embedding Luau, some runtimes may choose never to, particularly those whose module hierarchies don't look like traditional filesystem navigation. Projects targeting those runtimes will need to arrange their modules some other way, such as copying them into place on disk, with the identity issues described in Motivation.

## Alternatives

### The original `.modules` proposal

This RFC is a revision of [#265](https://github.com/luau-lang/rfcs/pull/265). The motivation for this revision is to both address technical challenges the original RFC raised (ensuring the Luau abstract library remains stateless and unopinionated) as well as complexity and ergonomic issues presented by community reviewers. Discussion of those issues can be found on the original PR.

One capability this revision gives up is attaching arbitrary configuration to parts of the module hierarchy, such as disabling lints for installed packages. Mounts carry aliases only; any other configuration comes from a mount's own `.config`.

### Mounts in `.config` ([#268](https://github.com/luau-lang/rfcs/pull/268))

#268 proposed adding a `mounts` option to `.config` files. This RFC shares its name and its core idea: placing a source into the abstract module hierarchy with its own identity, with the links between mounts defined outside of the shared source, so that deduplicated modules can each carry different aliases.

However, the mounts RFC requires the abstract library to thread state through navigation to maintain a stack of which mounts you have navigated into, which changes the shape of the abstract library from its current stateless and unopinionated shape, and introduces a significant amount of complexity into path resolution. The Luau team have [expressed discomfort](https://github.com/luau-lang/rfcs/pull/268#issuecomment-6529824803) at this.

By contrast, `.mounts.json` keeps mounts out of Luau's require resolution entirely. Because every mount is isolated, navigation never needs to remember which mounts it has passed through, and embedders that don't adopt `.mounts.json` are unaffected.

### Configuring mounts with `.config`

`.mounts.json` is _external configuration_, while `.config.luau` is _internal configuration_. The distinction between the two is: `.config` lives inside the location it configures aliases for, which makes it incompatible with deduplication, while `.mounts.json` lives outside of the hierarchies it configures for code to jump into, which means links between hierarchies are defined in a location deduplication won't tamper with, allowing for each instance of a deduplicated module to carry different aliases. This is what was previously referred to as ["category (a)" and "category (b)"](https://github.com/luau-lang/rfcs/pull/265#issuecomment-6457484521) solutions in the original PR threads.

### Using Luau instead of JSON

`.mounts.json` could instead be a Luau file, consistent with `.config.luau`. However, a Luau file can contain control flow and arbitrary logic, so it can't be machine edited in parts; a tool would have to evaluate it and overwrite it from scratch, losing anything the author wrote by hand.

A middle ground would be to introduce a Luau data format, relating to Luau the way JSON relates to JS: a machine-writable subset of Luau with no control flow, which tools could parse and edit safely. That would deserve its own proposal, and this RFC could adopt it once it exists.
