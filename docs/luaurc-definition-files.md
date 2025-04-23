# Definition Files in .luaurc Configs

## Summary

Implement a field in the `.luaurc` configuration file named "definitionFiles", which accepts an array of paths to ".d.luau" files (aka, declaration files). These definition files would be passed into the typechecker for any luau file under that configuration.

## Motivation

As cross-runtime code becomes increasingly common, the model which language servers use today (pass definition files into the API) is proving to not scale. Projects with multiple runtime targets suffer from bad editor experience, and great difficulty in getting types to be correct.

## Design

This is the bulk of the proposal. Explain the design in enough detail for somebody familiar with the language to understand, and include examples of how the feature is used.

## Drawbacks

This brings declaration files to be in a much more user-facing spot, when definition files are not meant to be user-facing.

## Alternatives

What other designs have been considered? What is the impact of not doing this?
