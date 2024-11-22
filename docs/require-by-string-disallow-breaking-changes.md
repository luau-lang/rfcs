# Disallow Amendments to Require By String syntax if They Break Compatibility

## Summary

The topic of Require By String has been the subject of multiple RFCs:

- [Require by String with Relative Paths](./new-require-by-string-semantics.md)
- [Require by String with Aliases](./require-by-string-aliases.md)
- [Amended Require Syntax and Resolution Semantics](./amended-require-resolution.md)

At the time of writing, there are currently multiple RFCs that _also_ amend the Require By String RFC. This RFC exists to ensure further amendments do not break compatibility.

## Motivation

Stability is an important guarantee for languages. While the details of Luau's require are technically implementor defined (Luau as-is does not come with any way to load other modules from user code), the semantics are not. As a result, multiple implementors (Lune, Luau's CLI, Luau LSP, DarkLua) have adopted the require by string RFCs, or have plans to adopt them.

As a result, the expectation is that code will be written that matches the semantics defined by these RFCs. Any RFCs that make breaking changes to the behavior of Require By String therefore have one of two consequences:

- User code will cease to function through no fault of the author
- Tools like Lune will diverge from the semantics of the RFCs

Both of these are considered undesirable, so this RFC exists to prevent breaking changes.

## Design

This RFC contains no language features, so the design is simple. Instead, accepting this RFC would be a promise.

The promise is: no RFCs will be accepted that will amend the Require By String RFC in a way that is breaking for users.

This does not:

- Prevent RFCs from amending Require By String semantics in a compatible way
- Prevent RFCs for new require semantics
- Prevent RFCs for unrelated features

This does:

- Prevent breaking the stability contract between Luau and its users

## Drawbacks

If a serious mistake is made in the design of require by string, this RFC would make it difficult if not impossible to correct. However, this is already true with other language features: mistakes happen, and Luau has to live with them just like any other language. In particular, it's the responsibility of language maintainers to not allow poorly designed or thought out RFCs into the language to begin with.

## Alternatives

This RFC is perhaps too restrictive and instead trust should be placed on the Luau maintainers to not break compatibility. However, the Luau team has already indicated a willingness to break compatibility if it's important. Since the definition of important is obviously not written down (it's impossible to know in advance what emergencies may arise), it may one day include the require by string RFC, which leads us to the same situation this RFC seeks to avoid.

The impact of not doing this is to simply allow breaking changes to occur as they're decided upon. This harms the stability of tools and user code and could easily lead to a distrust of Luau, as things will no longer seem stable. This is particularly impactful with require by string as it's already been amended multiple times, and users have had to change in response multiple times.
