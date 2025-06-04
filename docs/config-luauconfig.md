# Support Luau-syntax configuration files

**Status**: Not implemented

## Summary

Introduce `.luauconfig`: a Luau-syntax file used to configure analysis and runtime behavior.

## Motivation

We currently support analysis and runtime configuration in `.luaurc` files, which use a JSON-like syntax ([RFC](https://rfcs.luau.org/config-luaurc)).

Under the current design, the burden of providing JSON-editing support is placed on anyone embedding Luau, even if their environment is not naturally suited for JSON configuration.
Ideally, embedding Luau should not require implementers to build or maintain specialized tooling for editing JSON-based configuration files.

## Design

We introduce a new configuration file called `.luauconfig`.

### Syntax

An example `.luauconfig` is given below.

```luau
return {
    languageMode = "nonstrict",
    lint = {
        ["*"] = true,
        LocalUnused = false
    },
    lintErrors = true,
    typeErrors = true,
    globals = {"expect"},
    aliases = {
        src = "./src"
    }
}
```

The file must return a Luau table of the following `LuauConfig` type:
```luau
type LanguageMode = "strict" | "nonstrict" | "nocheck"

type LintWarning =
    "*"
    | "UnknownGlobal"
    | "DeprecatedGlobal"
    | "GlobalUsedAsLocal"
    | "LocalShadow"
    | "SameLineStatement"
    | "MultiLineStatement"
    | "LocalUnused"
    | "FunctionUnused"
    | "ImportUnused"
    | "BuiltinGlobalWrite"
    | "PlaceholderRead"
    | "UnreachableCode"
    | "UnknownType"
    | "ForRange"
    | "UnbalancedAssignment"
    | "ImplicitReturn"
    | "DuplicateLocal"
    | "FormatString"
    | "TableLiteral"
    | "UninitializedLocal"
    | "DuplicateFunction"
    | "DeprecatedApi"
    | "TableOperations"
    | "DuplicateCondition"
    | "MisleadingAndOr"
    | "CommentDirective"
    | "IntegerParsing"
    | "ComparisonPrecedence"
    | "RedundantNativeAttribute"

type LuauConfig = {
    languageMode: LanguageMode?,
    lint: { [LintWarning]: boolean }?,
    lintErrors: boolean?,
    typeErrors: boolean?,
    globals: { string }?,
    aliases: { [string]: string }?,
}
```

If configuration fields are not provided, `.luauconfig` uses the same default behavior as `.luaurc`.

### Search semantics

The search semantics of `.luauconfig` files are the same as `.luaurc`'s (adapted from old RFC):

> For a given `.luau` or `.lua` script, Luau will search for `.luaurc` files starting from the folder that the script is in; all files in the ancestry chain will be parsed and their configuration applied.
> When multiple configuration files exist throughout the ancestry chain, configurations closer to the script override those in higher-level directories.

However, if both `.luaurc` and `.luauconfig` are present in a directory, only the `.luauconfig` file is used.
Configuration files are not merged; at most one configuration file is recognized per directory.

### Configuration extraction

Unlike `.luaurc` files, `.luauconfig` files can contain runtime constructs like variables, functions, and loops.
To extract configuration from a `.luauconfig` file, its contents are executed in an isolated Luau VM with only the standard libraries enabled (`luaL_openlibs`).
This approach ensures that configuration files can leverage Luau features while maintaining a sandboxed execution environment.

#### Timing out

Executing arbitrarily complex Luau code requires us to guard against excessive computation:
```luau
-- .luauconfig with an infinite loop
return (function()
    while true do
    end
end)()
```

For analysis-time configuration extraction, we adopt the same behavior described in the [user-defined type functions RFC](https://rfcs.luau.org/user-defined-type-functions.html): we simply respect the time limit that the embedding context provides through the `Frontend` API.

At runtime, the configuration extraction timeout is set to two seconds by default.
This duration is generally sufficient for typical configuration logic, while being short enough to prevent accidental or malicious long-running scripts from impacting performance.
For more fine-grained control, however, embedders are free to override this at build-time by defining the `LUAUCONFIG_RUNTIME_TIMEOUT_SECONDS` macro.

## Drawbacks

**Complexity**: Allowing arbitrary Luau code in configuration files increases the complexity of parsing and validating configurations, which might make debugging more difficult.

**Multiple formats**: There might be some confusion surrounding the two different configuration files.
Most of this can be mitigated by (1) editor tooling, and (2) embedded contexts will likely choose one to support, not both.

## Alternatives

**Restrict syntax**: Limit `.luauconfig` files to only contain table literals and forbid functions, loops, and other runtime constructs to reduce complexity.
Instead of spawning a new Luau VM to "parse" the file, we could simply spin up a Luau parser.

**Improve support for `.luaurc` files**: We already provide an autocomplete API for Luau-syntax files and could expose an API that facilitates editing `.luaurc` files.
