# Definition Files in .luaurc Configs

## Summary

Implement a field in the `.luaurc` configuration file named "definitionFiles", which accepts an array of paths to ".d.luau" files (aka, declaration files). These definition files would be passed into the typechecker for any luau file under that configuration.

## Motivation

As cross-runtime code becomes increasingly common, the model which language servers use today (pass definition files into the API) is proving to not scale. Projects with multiple runtime targets suffer from bad editor experience, and great difficulty in getting types to be correct.

## Design

When analyzing a file, include all of the files specified in the `declarationFiles` field in analyzation. This supercedes the `globals` field, which should be deprecated.

## Drawbacks

This brings declaration files to be in a much more user-facing spot, when definition files are not meant to be user-facing. This could cause confusion, as definition files are currently largely undocumented and have poor syntax.

## Alternatives

TODO
