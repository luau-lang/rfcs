# Stricter utf8 library validation

**Status**: Implemented

## Summary

`utf8.len` and other functions in UTF-8 library will do more rigorous validation of UTF-8 input, rejecting UTF-16 surrogates encoded into UTF-8.

## Motivation

We use the UTF-8 library `utf8` from Lua 5.3. The implementation of this library mostly correctly validates UTF-8 input and either throws errors (e.g. from `utf8.codes`) or returns `nil` (for `utf8.len`) for invalid input.

However, certain invalid UTF-8 sequences are treated as valid. This specifically includes surrogate characters encoded in UTF-8; for example, the string `"\237\160\128"` represents an attempt to encode the surrogate `0xD800` into UTF-8.
This is impossible to do in UTF-8; Lua 5.3 accepts this (`utf8.len` returns 1), and so does Luau.

This creates issues for any other API that correctly validates UTF-8. For example, Roblox extends Luau with a few extra functions, like `utf8.nfcnormalize`, that perform UTF-8 validation correctly and reject the aforementioned string.
As a result, due to Roblox extensions, the `utf8` library exposed in Roblox is inconsistent (this is also a problem in other Roblox-specific APIs like DataStores that reject invalid UTF-8 inputs that `utf8.len` accepts).
We would also expect that in other environments, extra functions that handle UTF-8 inputs and validate them do the validation properly.

Lua 5.4 fixes this by changing the default behavior of `utf8.len` and other UTF-8 functions to error when surrogates are present. It also fixes an issue in `utf8.codes` that failed to properly reject overlong UTF-8 sequences in certain cases; Luau already contains a fix to this issue.

## Design

All functions from Luau `utf8` library will correctly validate UTF-8 and reject inputs that have surrogates. The functions will use their existing error handling strategy: `utf8.len` will return `nil` followed by the byte offset, `utf8.codepoint` and `utf8.codes` will error.

Any code using these functions should already be prepared for handling errors: `utf8` functions will already do a validation of UTF-8 input, the validation was just not fully comprehensive. It is expected that this will not cause any significant compatibility risk, because many other functions already reject surrogates and the use of `utf8.` function family in general is rare.

Notably, this is a *partial* implementation of Lua 5.4 behavior. In addition to making the default behavior stricter, Lua 5.4 *relaxes* `utf8.char` (allowing it to produce "codepoints" that exceed 0x10FFFF and are invalid per Unicode standard) unconditionally, and also provides an optional argument `lax` to a few functions like `utf8.len` that allows to accept both surrogates and out-of-range (larger than `0x10FFFF`) codepoints.

We do not propose to adopt the full Lua 5.4 behavior or introduce the new `lax` argument for the following reasons:

- Based on the analysis of all packages published to `luarocks`, we conclude that `lax` argument is never used outside of Lua's own tests. As such, it's questionable whether we *need* to give a way to relax the checks.
- The behavior changes regardless due to the parameter defaults (`lax` defaults to `false`). We agree with this choice but conclude that the compatibility risk, however small, exists regardless of the argument's presence.
- Because full 5.4 semantics include both making behavior stricter by default, and also significantly relaxing behavior by allowing out-of-range characters (unconditionally for `utf8.char`, and conditionally on `lax` for other `utf8` functions), they allow easier production of invalid UTF-8 sequences.

It is likely that 5.4 semantics change is motivated by only having two types of behavior: "strict" (proposed here) and "lax" (supporting a variant of UTF-8 that allows encoding any codepoint value). Because it is always possible to add the "lax" behavior later, and we do not want to establish the precedent and require this addition to all other functions (for example, what should `nfcnormalize` do for surrogates?), we only specify the "strict" behavior here. The "lax" behavior could be added by a separate RFC if there's significant demand for it; that would be fully backwards compatible.

> After this change, in Roblox strings that are correctly validated by `utf8.len` should also work properly with `utf8.nfcnormalize`, `utf8.graphemes`, DataStore APIs and other functions that validate UTF-8 input.

## Drawbacks

Because this technically changes behavior, due to Hyrum's law we might break some code that *really* wants to encode surrogates in UTF-8 despite the fact that only a couple functions support this.

Because this doesn't fully align behavior with Lua 5.4, it is possible that we will get programs in the future that want the full Lua 5.4 treatment (allowing `lax`). It is likely that this will happen because of larger codepoints for some odd custom encoding reasons though, not for surrogates specifically, and if it does we can add it later.

## Alternatives

Other than doing nothing, we could implement full Lua 5.4 behavior (with `lax` argument and relaxed range checks in some existing functions). This leaves the question as to what the behavior of Roblox-specific utf8 extensions like `nfcnormalize` or `graphemes` should be, as they would become inconsistent not just in implementation but also in interface without further changes.
