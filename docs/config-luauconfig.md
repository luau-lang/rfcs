# Support Luau-syntax configuration files

**Status**: Implemented

## Summary

Introduce `.config.luau`: a Luau-syntax file used to configure analysis and runtime behavior.

## Motivation

We currently support analysis and runtime configuration in `.luaurc` files, which use a JSON-like syntax ([RFC](https://rfcs.luau.org/config-luaurc)).

Under the current design, the burden of providing JSON-editing support is placed on anyone embedding Luau, even if their environment is not naturally suited for JSON configuration.
Ideally, embedding Luau should not require implementers to build or maintain specialized tooling for editing JSON-based configuration files.

## Design

We introduce a new configuration file called `.config.luau`.

### Syntax

An example `.config.luau` is given below.

```luau
return {
    luau = {
        languagemode = "nonstrict",
        lint = {
            ["*"] = true,
            LocalUnused = false
        },
        linterrors = true,
        typeerrors = true,
        globals = {"expect"},
        aliases = {
            src = "./src"
        }
    }
}
```

The file must return a Luau table of the following `Config` type (facilitated by type checking and autocomplete support):
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
    languagemode: LanguageMode?,
    lint: { [LintWarning]: boolean }?,
    linterrors: boolean?,
    typeerrors: boolean?,
    globals: { string }?,
    aliases: { [string]: string }?,
}

type Config = {
    luau: LuauConfig?
}
```

If configuration fields are not provided, `.config.luau` has the same default behavior as `.luaurc`.

### Search semantics

The search semantics of `.config.luau` files are the same as `.luaurc`'s (adapted from a previous RFC):

> For a given `.luau` or `.lua` script, Luau will search for `.luaurc` files starting from the folder that the script is in; all files in the ancestry chain will be parsed and their configuration applied.
> When multiple configuration files exist throughout the ancestry chain, configurations closer to the script override those in higher-level directories.

If both `.luaurc` and `.config.luau` are present in a directory, an error is thrown.

### Configuration extraction

Unlike `.luaurc` files, `.config.luau` files can contain runtime constructs like variables, functions, and loops.
To extract configuration from a `.config.luau` file, its contents are executed in an isolated Luau VM with only the standard libraries enabled (`luaL_openlibs`).
This approach ensures that configuration files can leverage Luau features while maintaining a sandboxed execution environment.
As a consequence, `.config.luau` files cannot require other scripts, as `require` is not included in the standard libraries.

#### Timing out

Executing arbitrarily complex Luau code requires us to guard against excessive computation:
```luau
-- .config.luau with an infinite loop
return (function()
    while true do
    end
end)()
```

For analysis-time configuration extraction, we adopt the same behavior described in the [user-defined type functions RFC](https://rfcs.luau.org/user-defined-type-functions.html): we simply respect the time limit that the embedding context provides through the `Frontend` API.

At runtime, the configuration extraction timeout for each `.config.luau` file is set to two seconds by default.
This duration is generally sufficient for typical configuration logic while being short enough to prevent accidental or malicious long-running scripts from impacting performance.
For more fine-grained control, however, embedders will be free to override this.
That is to say, we do not wish to prescribe a specific limit in general, but note that having a limit in place is important.

### Interaction with `require`

Despite `.config.luau` being a `.luau` file, it is not recognized as a module by `require` at runtime.
Concretely, the source file `foo.luau` cannot extract the contents of its sibling `.config.luau` file with the expression `require("./.config")`.

## Drawbacks

**Complexity**: Allowing arbitrary Luau code in configuration files increases the complexity of parsing and validating configurations, which might make debugging more difficult.
Allowing runtime constructs also makes automated edits more difficult to support.

**Multiple formats**: There might be some confusion surrounding the two different configuration files.
Most of this can be mitigated by (1) editor tooling, and (2) embedded contexts will likely choose one to support, not both.

## Alternatives

**Restrict syntax**: Limit `.config.luau` files to only contain table literals and forbid functions, loops, and other runtime constructs to reduce complexity.
Instead of spawning a new Luau VM to "parse" the file, we could simply spin up a Luau parser.

**Improve support for `.luaurc` files**: We already provide an autocomplete API for Luau-syntax files and could expose an API that facilitates editing `.luaurc` files.
