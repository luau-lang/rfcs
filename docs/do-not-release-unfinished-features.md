# Do Not Release Unfinished Features

## Summary

Luau should not ship incomplete or unusable features that are not ready for users, even if part of the feature is ready.

## Motivation

Recently, Luau has released the vector primitive type and vector standard library. However, only the VM part of the vector was ready for release. The analysis vector primitive is, at the time of writing, completely unusable. Releasing unfinished and unusable features can lead to confusion and frustration among users and will ultimately turn people away from the language. It also harms tooling, like Luau-LSP, which struggles to support these features which are not ready for users.

## Design

This RFC adds no language features, and is instead a change to how Luau releases features. Under this RFC, Luau will not release features that are not ready in every sense of the word.

As most users get their FFlags from Roblox, and Luau is a Roblox product, for the purposes of this RFC, Roblox enabling a FFlag is the same as Luau releasing a feature.

## Drawbacks

This RFC has the drawback of slowing down the release of new features. However, this is necessary to ensure that what is released is ready for users.

## Alternatives

Instead of adopting this RFC, trust can be placed in the Luau team to not release features prematurely in the future.
