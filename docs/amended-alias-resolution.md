# Amended Alias Syntax and Resolution Semantics

## Summary

We need to finalize a syntax for aliases in require statements and determine intuitive resolution semantics as we prepare to build a package management system.

## Motivation

As we prepare to build an official Luau package management system, we need unambiguous alias functionality.

In an [old RFC for require-by-string](https://github.com/luau-lang/luau/pull/969), we removed the `@` prefix from aliases altogether and opted for the following design:
the path `require("my/module")` is resolved by first checking if `my` is an alias defined in a `.luaurc` file.
If so, we perform a string substitution on `my`.
If not, we try to resolve `my/module` relative to the requiring file.
This is identical to approach (3a) below.

This design changed in the next few months and became what we now have in [require-by-string-aliases.md](require-by-string-aliases.md):
when requiring an alias, it must be prepended with the `@` prefix.
Otherwise, we will assume the given string does not contain an alias and will resolve it relative to the requiring file.
This is identical to approach (1a) below.

There have been disagreements about which approach is more appropriate, so this RFC will serve as a way to document the ongoing discussion and the final decision we make.

## Possible Designs and Drawbacks

Below, we have multiple different options for syntax and resolution semantics.
For each design, assume that the following `.luaurc` file is defined:
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

#### (1a) Make aliases explicit with `@` prefix

We make usage of the `@` prefix necessary in aliased require statements.
This means that any unprefixed string is always treated as an unaliased path.
```luau
-- Requires ./libs/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

The main drawback of this approach is that projects with many _external_ dependencies will have many `@` symbols appearing within require statements.

This is similar to [Lune's approach](https://lune-org.github.io/docs/getting-started/2-introduction/8-modules#file-require-statements) and is identical to the approach currently described in [require-by-string-aliases.md](require-by-string-aliases.md).

#### (1b) Make relative paths explicit with `./` prefix

We make usage of the `./` prefix necessary in relative require statements.
This means that any unprefixed string is always treated as an aliased path.
```luau
-- Requires /My/Libraries/Directory/dependency.luau
require("libs/dependency")

-- Requires ./libs/dependency.luau
require("./libs/dependency")

-- Requires /My/Libraries/Directory/dependency.luau
require("@libs/dependency")
```

The main drawback of this approach is that projects with many _internal_ dependencies will have many `./` symbols appearing within require statements.

This is similar to [darklua's approach](https://darklua.com/docs/path-require-mode/).

#### (1c) All explicit prefixes necessary

An alternative approach is to combine approaches (1a) and (1b) and always require prefixes for disambiguation.
In this scenario, any unprefixed path will _always_ result in an error.
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

The main drawback of this approach is that it is not backwards compatible with any existing code that uses unprefixed require statements and may result in code that looks cluttered.

## Alternatives

Below are options that were proposed in an earlier version of this RFC, with their numbered labels preserved.
These approaches were deemed undesirable during discussion and are no longer being considered.

### (2) Strongly recommend explicit prefixes

We don't make the `@`, `./`, or `../` prefixes necessary like in approach (1c), but we strongly recommend them.

This can be accomplished by allowing unprefixed paths but throwing an error if they resolve to multiple files.
There is a performance cost to this: currently, if we successfully match a given path to a file, we stop searching for other matches.
If this approach were implemented, we would need to search for at most two matching files in case we need to throw an error.
In the case of multiple nested directories with their own `.luaurc` files and `paths` arrays, this search could take linear time with respect to the total number of paths defined, in the worst case.

Of course, developers could use the `@`, `./`, or `../` prefixes to explicitly specify a search type and eliminate this performance cost.
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