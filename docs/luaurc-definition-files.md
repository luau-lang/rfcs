# Definition Files in .luaurc Configs

## Summary

Implement a field in the `.luaurc` configuration file named "definitionFiles", which accepts an array of paths to ".d.luau" files (aka, definition files). These definition files would be passed into the typechecker for any luau file under that configuration.

## Motivation

As cross-runtime code becomes increasingly common, the model which language servers use today (pass definition files into the API) is proving to not scale. Projects with multiple runtime targets suffer from bad editor experience, and great difficulty in getting types to be correct. This is because you cannot manage runtime types on a per-folder basis, which makes monorepos difficult, as types from, for example, the Lute runtime, will leak into Roblox files.

Taking advantage of `.luaurc`'s inheriting behavior allows for granular control over definition files. This could be used for something like specifying testing framework globals used within a `tests` directory, when the project is under a runtime with definition files already.

## Design

When analyzing a file, include all of the files specified in the `definitionFiles` field in analyzation. This supercedes the `globals` field, which is deprecated under this proposal.

If a specified path is relative, then it should be resolved from the path of the luaurc file specifying it.

Example usage:

```
{
	"definitionFiles": [ "~/.lute/typedefs/0.1.0/globals.d.luau" ]
}
```

## Drawbacks

This brings definition files to be in a much more user-facing spot, when definition files are not meant to be user-facing. This could cause confusion, as definition files are currently largely undocumented and have poor syntax.

## Alternatives

This isn't truly necessary, and relying on code editors to provide type definition files does work.
