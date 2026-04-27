# Syntax editions

## Summary

Introduce syntactic editions to allow for backward incompatible changes to be added to the language without breaking user code as a result.

## Motivation

Luau strives to be as backwards compatible as possible when implementing new features - the requirement for code that was written 10+ years ago to still function properly today is one of Luau's main goals.

However, these restrictions prevent the acceptance of many desired feature additions for a multitude of reasons, mostly because of quirks inherited from Lua 5.1. Backwards compatibly prevents the introduction of optimal syntax additions and encourages proposers to search for "funky" alternatives which often tend to be less intuitive (e.g. using keywords that are already reserved but read wrong) for users. This leads to unfortunate syntax proposals which try to wiggle their way round these previously set restrictions, producing suboptimal syntax as a result, when, usually, a better syntax is obviously clear.

As a (non-exhaustive) example of the many quirks inherited from Lua 5.1 that introduce ambiguities:

- Statements don't require termination, [which required an RFC to disallow such ambiguities](https://rfcs.luau.org/disallow-proposals-leading-to-ambiguity-in-grammar.html).
- Multiline string syntax, which prevents a separate array-like syntax from being introduced (which itself lowers the chances of table destructing being added)
- String and table call syntax (e.g. `foo {}` and `bar ""`), which, in most cases, prevents contextual keywords from being added (e.g. `match <expr> ...` is ambiguous without infinite lookahead because of `match "foo"` already being a call)

This RFC does not propose that any of these features be removed or changed; it simply provides a mechanism for breaking changes to be introduced safely, allowing for the most optimal syntax. In majority of cases, new syntax additions should not require a new edition but instead be added to an existing one.

## Design

This RFC proposes the introduction of editions: each edition is a set of backwards incompatible parsing changes, uniquely identified by the year it was proposed. Those parsing changes only apply if a user has specifically opted into the new edition, meaning that new breaking syntax can be added without actually breaking any user code.

Developers can specify which edition they would like to use in two ways:

- per file using the `--!edition` comment directive (e.g. `--!edition 2026` to use the 2026 edition for the current file)
- per workspace using `.config.luau` or `.luaurc` under the `"edition"` field

If both are provided, the comment directive should take priority. When no edition is specified by the user, Luau should default to the 2020 edition (that is, the syntax as it stands today).

> Important note:
>
> Editions, as proposed here in this RFC, are completely syntactical: they still target the same bytecode and are still run by the same VM. This means that files can still interact (i.e. be required) even if they each use a different edition: for example, if file A specified the edition 2026 and file B specified 2020, file A could require file B without issue and vice versa - editions do not affect the compiled code.
>
> Additionally, a quirk of the semantics proposed is that `.config.luau`'s edition must always be specified with a comment directive, otherwise it will default to edition `"2020"` as it cannot tell itself that it needs to parse a newer edition and also cannot share a home with `.luaurc`.

Editions are proposed using the RFC process, and their *Design* section should specify the set of required backwards incompatible changes that will be included and which RFCs require those breaking changes. They are uniquely identified by year, meaning, at most, one edition can be released per year (this, however, is unideal and should be avoided). New non-breaking syntax additions can be added to the current edition at any time (as per the RFC process), however, no breaking syntax additions should be added to an edition after it is released.

Once superseded by a newer edition, the preceded edition becomes completely immutable: no new syntax additions (non-breaking or otherwise) should be added to that edition - only issues which are clearly bugs in the implementation should be changed.

## Drawbacks

- The Luau team now need to support many different versions of syntax for the same language.
- Syntax highlighting is now much more complex as it has to support multiple editions worth of syntax. For example, if multiline string syntax was changed, the original multiline string syntax would still have to be highlighted, especially for less workspace aware software like GitHub.
- More configuration makes interacting and debugging outside code (e.g. library code) more confusing: one's mind must mentally switch between versions... "which syntax can I use in this module vs this one?"
- Because of immutability, older editions can not introduce new syntax to leverage newer VM features, meaning that only newer editions benefit from newer VM features.

## Alternatives

### Mutable editions

Instead of making superseded editions immutable, older editions could be allowed to have newer syntax backported to them from the current edition, meaning that a new syntax addition could be added to an older edition and affect all the editions after it. This means that, when new features are added to the VM, developers who are still on older editions can leverage new syntax with having to migrate all their legacy code.

However, the assumptions of linearity users can make from immutable editions described in the *Design* section leads to less confusion for users: if they see a year greater than the last, they can assume that newer features will be present - adding mutable editions blurs the line between which features are and aren't available in a given edition whereas immutable editions make that explicit. The granularity of defining an edition for a specific module should also benefit developers who want to gradually migrate their code to newer syntax as modules on newer editions can still be required by older ones.

### Do nothing

Do not implement editions and continue adding only backwards compatible syntax additions to the language.

The reasons outlined in *Motivation* should be enough to persuade again this: those restrictions lead to suboptimal syntax where, in most cases, a cleaner, more universally accepted syntax is clear. Finding the language in a position where highly requested features either can't be introduced or are required to have unfamiliar, suboptimal syntax is extremely unfortunate, especially when there is a clear way to avoid it.
