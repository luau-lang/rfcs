# Amended Require Resolution Semantics and Syntax as it Relates to `init` Files

## Summary

This proposes extended rules to require resolution with respect to `init` files.

## Motivation

An issue was discovered with previous semantics for require resolution when implemented in roblox, especially with the popular tool Rojo:

```
- folder/
  - init
  - module
```

With the above project, one could could require `module` from `init` with `./module` in the filesystem, but once Rojo or another tool had moved the project into Roblox, the require would fail because the structure would change into:

```
- folder/ (init)
  - module (module)
```

Due to this, the proper require would be `./folder/module`. This is inconsistent with the filesystem, and disregarding that, it is unintuitive. This proposal corrects this issue.

## Design

For the purposes of this RFC, file extensions will be ommitted. Per previous RFCs, both `luau` and `lua` are valid extensions, but having both at the same path is an error due to ambiguity.

As written in previous RFCs, if a require string resolves to a directory instead of a file, then we will attempt to require a file in that directory named `init`. This RFC extends that rule to disallow requiring files named `init` directly.

This proposal changes require resolution when requiring from an `init` file. Relative requires from an `init` file will be resolved relative to the directory containing the `init` file, not the `init` file itself.

This makes requiring a child of a folder from the folder's `init` file unintuitive and inconvenient, `./{self}/child` instead of what currently is `./child`. The solution to this is the child require prefix, `:`. This prefix allows requiring a child of the current directory from an `init` file to be done with `:child`.

As an example, consider the following filesystem project structure and table of require resolutions:

```
- sibling
- folder/
  - init
  - module
  - subfolder/
    - init
	- submodule
```

| From                         | To                           | Path                   |
| ---------------------------- | ---------------------------- | ---------------------- |
| `sibling`                    | `folder/init`                | `./folder`             |
| `folder/init`                | `sibling`                    | `./sibling`            |
| `folder/init`                | `folder/module`              | `:module`              |
| `folder/init`                | `folder/subfolder/init`      | `:subfolder`           |
| `folder/init`                | `folder/subfolder/submodule` | `:subfolder/submodule` |
| `folder/module`              | `folder/init`                | `.`                    |
| `folder/module`              | `folder/subfolder/init`      | `./subfolder`          |
| `folder/subfolder/init`      | `sibling`                    | `../sibling`           |
| `folder/subfolder/init`      | `folder/init`                | `.`                    |
| `folder/subfolder/init`      | `folder/module`              | `./module`             |
| `folder/subfolder/init`      | `folder/subfolder/submodule` | `:submodule`           |
| `folder/subfolder/submodule` | `folder/subfolder/init`      | `.`                    |
| `folder/subfolder/submodule` | `folder/init`                | `..`                   |

## Drawbacks

This proposal has some significant drawbacks that deserve consideration:

- Some users have already migrated to the require resolution semantics that this proposal changes. However, as these require semantics have not made it into Luau's largest user, Roblox, or Luau's primary language server, Luau-LSP, these users are few.
- This proposal changes require resolution semantics to deviate from the filesystem in a significant way. This will be unintuitive to some users, although it is important to note that other languages like Rust have similar import semantics.

## Alternatives

- As it relates to requiring children of a folder from the folder's `init` file, the child require prefix could be changed to almost anything. Some notable examples include `@mod`, `#`, and `#mod`.
- If this proposal is not accepted and the status-quoe is maintained, then projects using Rojo will have to bear significant changes and inconvenience if they wish to benefit from new require resolution semantics. Roblox projects not using Rojo may not have such severe issues, but a common pattern will be made more annoying.
