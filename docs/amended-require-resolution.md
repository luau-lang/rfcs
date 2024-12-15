# Amended Require Syntax and Resolution Semantics

**Status**: Implemented

## Summary

We need to finalize a syntax for aliases in require expressions and determine intuitive resolution semantics as we prepare to build a package management system.

## Motivation

In an [old RFC for require-by-string](https://github.com/luau-lang/luau/pull/969), we removed the `@` prefix from aliases altogether and opted for the following design:
the path `require("my/module")` is resolved by first checking if `my` is an alias defined in a `.luaurc` file.
If so, we perform a string substitution on `my`.
If not, we try to resolve `my/module` relative to the requiring file.
This is identical to approach (3a) below.

This design changed in the next few months and became what we now have in [require-by-string-aliases.md](require-by-string-aliases.md):
when requiring an alias, it must be prepended with the `@` prefix.
Otherwise, we will assume the given string does not contain an alias and will resolve it relative to the requiring file.
This is identical to approach (1a) below.

As we prepare to build an official Luau package management system, we need unambiguous alias functionality.
This RFC documents the final design decision made after discussion.

Note: The numbered labels below are preserved from an earlier version of this document.
Discussion in the [PR](https://github.com/luau-lang/rfcs/pull/56) refers to these approaches by their labels, so they have been left as-is in this document.

## Design

### Context

Assume that the following `.luaurc` file is defined:
```json
{
    "aliases": {
        "libs": "/My/Libraries/Directory"
    }
}
```
Additionally, assume that we have a file named `dependency.luau` located in these two locations:
1. `./libs/dependency.luau`
2. `/My/Libraries/Directory/dependency.luau`

### (1) Make explicit prefixes necessary

#### (1c) All explicit prefixes necessary

We will always require prefixes for disambiguation.
In other words, any unprefixed path will _always_ result in an error.
```luau
-- Error: must have either "@", "./", or "../" prefix
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

A benefit of this approach is that the role of autocomplete becomes fairly well-defined.
When `require("` is typed, autocomplete has only three options to display: the `@`, `./`, and `../` prefixes.
After a prefix has been typed, autocomplete will either display defined aliases or local modules, depending on the prefix.
Approaches (1a) and (1b) would be less clear about this, as `require("` could be followed by a string component or one of the three prefixes.

An additional benefit of this approach is that it is forwards-compatible with both (1a) and (1b), which are now in Alternatives.
We effectively reserve unprefixed paths with this approach and can support either (1a) or (1b) further down the road.
After a package manager has been implemented, we will likely have a better sense for which option makes the most sense: mapping unprefixed paths to local modules, to aliased modules, or to nothing (keeping unprefixed paths unsupported).

### Removing `paths`

As part of the push to make require expressions more explicit, we will remove `paths`, as aliases defined in `aliases` serve a similar purpose and are more explicit.

### Throw an error if multiple files matched

Currently, we allow `require` to resolve to both `.luau` and `.lua` files, implicitly preferring `.luau` if both are present.
In line with this proposal's move toward explicit resolution semantics, this behavior will be revised.
We will continue to support `.luau` and `.lua` files, but we will throw an error if both are present.

We also currently support requiring a directory if it contains an `init.luau` or `init.lua` file.
If the name of a file matches the name of a sibling directory, we will now also throw an error to avoid ambiguity.

For example, calling `require("./module")` from `requirer.luau` would throw an error in each of these cases (non-exhaustive).

Conflict between `module.lua` and `module.luau`:
```
.
├── module.lua
├── module.luau
└── requirer.luau
```

Conflict between `module` directory and `module.luau`:
```
.
├── module
│   └── init.luau
├── module.luau
└── requirer.luau
```

## Drawbacks

The main drawback of this approach is that it is quite strict and not backwards-compatible with any existing code that uses unprefixed require expressions.
However, because of its forwards compatibility with approaches (1a) and (1b), it is possible for us to relax these restrictions in the future.

From an aesthetic point of view, this approach is slightly more cluttered than other approaches, as every path must begin with a prefix.

The removal of `paths` and changes to the implicit resolution order for different file types may also create concerns for backwards compatibility.

## Alternatives

### (1) Make explicit prefixes necessary

Although (1a) and (1b) are both explicit approaches, they take opposite stances on whether writing relative-path requires or aliased-path requires should be optimized for the developer.
Currently, relative-path requires are far more common, which fits well with approach (1a).
In the context of a package management system, however, we would expect aliased paths to be used more frequently, which fits better with approach (1b).

These approaches have been moved to Alternatives since they are both backwards-compatible with approach (1c).
Rather than making an arbitrary decision now, it makes more sense for us to choose (1c) now and reevaluate once package management is a reality.

#### (1a) Make aliases explicit with `@` prefix

We make usage of the `@` prefix necessary in aliased require expressions.
This means that any unprefixed string is always treated as an unaliased path.
```luau
-- Requires ./libs/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

The main drawback of this approach is that projects with many _external_ dependencies will have many `@` symbols appearing within require expressions.

This is similar to [Lune's approach](https://lune-org.github.io/docs/getting-started/2-introduction/8-modules#file-require-statements) and is identical to the approach currently described in [require-by-string-aliases.md](require-by-string-aliases.md).

#### (1b) Make relative paths explicit with `./` prefix

We make usage of the `./` prefix necessary in relative require expressions.
This means that any unprefixed string is always treated as an aliased path.
```luau
-- Requires /My/Libraries/Directory/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

The main drawback of this approach is that projects with many _internal_ dependencies will have many `./` symbols appearing within require expressions.

This is similar to [darklua's approach](https://darklua.com/docs/path-require-mode/).

### (2) Strongly recommend explicit prefixes

We don't make the `@` or `./` prefixes necessary like in approach (1c), but we strongly recommend them.

This can be accomplished by allowing unprefixed paths but throwing an error if they resolve to multiple files.
There is a performance cost to this: currently, if we successfully match a given path to a file, we stop searching for other matches.
If this approach were implemented, we would need to search for at most two matching files in case we need to throw an error.
In the case of multiple nested directories with their own `.luaurc` files and `paths` arrays, this search could take linear time with respect to the total number of paths defined, in the worst case.

Of course, developers could use the `@` or `./` prefixes to explicitly specify a search type and eliminate this performance cost.
```luau
-- Error: ./libs/dependency.luau exists and .luaurc defines a distinct "libs" alias
-- Would require ./libs/dependency.luau if "libs" alias did not exist
-- Would require /My/Libraries/Directory/dependency.luau if ./libs/dependency.luau did not exist
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

This is a compromise between approaches (1a) and (1b) that sacrifices performance for compatibility and readability.
Unfortunately, this approach could lead to unexpected behavior if users have globally defined `.luaurc` files that conflict with scripts that expect certain aliases to not be defined.

### (3) Use implicit resolution ordering to avoid `@` prefix

If we choose to remove the `@` prefix, we must define an implicit resolution ordering.
In essence, aliases must either represent overrides or fallbacks for relative path resolution.
These approaches are not being considered since implicit resolution ordering can be confusing, and people may disagree on which ordering is best.

#### (3a) Aliases as overrides

If aliases were overrides, we would have the following behavior.
This is identical to the [old RFC](https://github.com/luau-lang/luau/pull/969)'s behavior.
```luau
-- Requires /My/Libraries/Directory/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Error: there is no "@libs" alias in .luaurc
require("@libs/dependency")
```

If the `libs` alias did not exist, however, we would have the following:
```luau
-- Requires ./libs/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Error: there is no "@libs" alias in .luaurc
require("@libs/dependency")
```

#### (3b) Aliases as fallbacks

If aliases were fallbacks, we would have the following behavior.
```luau
-- Requires ./libs/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Error: there is no "@libs" alias in .luaurc
require("@libs/dependency")
```

If `./libs/dependency.luau` did not exist, however, we would have the following:
```luau
-- Requires /My/Libraries/Directory/dependency.luau
require("libs/dependency")

-- Error: does not exist
require("./libs/dependency")

-- Error: there is no "@libs" alias in .luaurc
require("@libs/dependency")
```