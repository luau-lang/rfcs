# Absolute Requires

## Summary

This proposal describes a new absolute syntax to support require-by-string in a format that can work for filesystems and embedded applications such as the Roblox platform, replacing the existing syntax we have today.

## Motivation

It should be possible for all users of Luau to write code that is capable of running, without modification, on many different platforms. Unfortunately, we identified an issue with the current syntax that would make it difficult for users of certain tools to adopt. As paths are relative to the file calling require, any tools that adjust the file hierarchy would also need to update require paths to match. While we know there may not be a perfect solution that works for all use-cases, we think the current syntax should be amended to support these cases or at least not require them to translate require paths in files.

## Design

### Overview

When calling require without a prefix, the path will be resolved as an absolute path where the root directory is defined as follows:

* If the file does not belong to a library, then the entry point's directory
* If the file belongs to a library, then the directory containing the library

For example, here's a simple file system:

```
├── folder
│   └── file.luau
├── module.luau
└── init.luau
```

And here's how you would navigate between files using the proposed syntax when init.luau is the entry point:

| **From**         | **To**           | **Path**    |
|------------------|------------------|-------------|
| init.luau        | module.luau      | module      |
| init.luau        | folder/file.luau | folder/file |
| module.luau      | folder/file.luau | folder/file |
| folder/file.luau | module.luau      | module      |

With this syntax, paths are the same no matter where a file is in a project, meaning they can be restructured by the user or with tools, without creating any issues.

### Libraries

While this RFC does not call out the exact definition of a library, there will be a way to designate a directory as such. Here is an example of a filesystem including one:

```
├── library (defined in some way)
│   ├── folder
│   │   └── file.luau
│   └── module.luau
└── init.luau
```

And here's how you would navigate between these files:

| **From**                 | **To**                   | **Path**       |
|--------------------------|--------------------------|----------------|
| init.luau                | library/module.luau      | library/module |
| library/module.luau      | library/folder/file.luau | folder/file    |
| library/folder/file.luau | library/module.luau      | module         |

Again, paths are still reasonably simple, but this time files in the library requires are relative to the root directory of the library.

### Init

As they can today, files called init.luau inside of a directory can be required by requiring the path to the directory they are in.

```
├── folder
│   └── init.luau
├── module.luau
└── init.luau
```

Given this file structure, it's possible to require folder/init.luau with require("folder").
However, one subtle change is that you won't be allowed to require init.luau directly. We're doing this for cases where files can also act as directories in certain environments.

### Aliases

There are no changes to aliases in this proposal.

### Comparison with Relative Paths

We will continue to support relative paths, however recommend updating to absolute paths to guarantee portability between different environments.

```
├── folder
│   └── init.luau
├── library (defined in some way)
│   ├── folder
│   │   └── file.luau
│   └── init.luau
├── module.luau
└── init.luau
```

| **From**          | **To**                   | **Relative Path**                               | **Absolute Path**    |
|-------------------|--------------------------|-------------------------------------------------|----------------------|
| init.luau         | module.luau              | ./module                                        | module               |
| init.luau         | folder/init.luau         | ./folder                                        | folder               |
| init.luau         | library/init.luau        | ./library                                       | library              |
| init.luau         | library/folder/file.luau | ./library/folder/file                           | library/folder/file  |
| module.luau       | folder/init.luau         | ./folder                                        | folder               |
| module.luau       | library/init.luau        | ./library                                       | library              |
| folder/init.luau  | module.luau              | ./../module ../module                           | module               |
| folder/init.luau  | library/folder/file.luau | ./../library/folder/file ../library/folder/file | library/folder/file  |
| library/init.luau | library/folder/file.luau | ./folder/file                                   | folder/file          |

As paths are relative to the root directory with the absolute syntax the only ones that change are those in files under a subdirectory. All other paths remain the same as the current syntax.

## Drawbacks

As relative paths are still available, we haven't completely solved the problem. Users could still opt to use them, leading to the issue mentioned above. However, there may be options (such as tooling) to guide users away from relative paths over time to help address this.

## Alternatives

We have already considered (and even approved) some alternatives to this approach, however there are drawbacks with each of them that ultimately led us in this direction.

* [Amended Require Syntax and Resolution Semantics](https://github.com/luau-lang/rfcs/pull/56)
* [Require by String (split into two RFCs)](https://github.com/luau-lang/luau/pull/969)
    * [Require by String with Aliases](https://github.com/luau-lang/rfcs/pull/7)
    * [New Require by String Semantics](https://github.com/luau-lang/rfcs/pull/6)
* [Amended Require Resolution Init](https://github.com/luau-lang/rfcs/pull/76)
