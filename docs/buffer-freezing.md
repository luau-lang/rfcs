# `buffer.freeze` and `buffer.isfrozen`

## Summary

Adds two new functions; `buffer.freeze(buf)` for creating a "read-only" buffer, and `buffer.isfrozen` to check if a
buffer is frozen.

## Motivation

Buffers are fully mutable, but this behavior might not always be desired for a user. There is currently no way to create
buffers which can be read, but not modified. This can result in lots of nasty safety issues when immutability is
desired, but not enforced; For example, a library passing a buffer to a series of user callbacks. This feature is useful
for sandboxing a memory buffer without abstracting it into a table with methods, or generally exposing immutable memory.

## Design

This RFC allows buffers to enter a "read-only" state by providing two new functions:

- `buffer.freeze(buf)`
- `buffer.isfrozen(buf)`

When a buffer is frozen, attempts to modify its data in any way will result in an error. Functions which do not modify
data (`buffer.len`, `buffer.read*`, etc.) will still work as expected.

## Drawbacks

- Likely introduces another branch to the VM's logic for the buffer library. Buffers are largely designed around
performance, so this is an important consideration. Modern CPUs with branch predictors can optimize for branches,
especially if someone never uses a frozen buffer. The larger concern is making VM/NCG/Runtime logic more complicated
when it comes to dealing with buffers, but the inverse argument can be made - immutable buffers could actually lead to
VM/NCG/Runtime performance optimizations.
- This increases the number of standard library functions, which naturally makes the language harder to "learn". More
importantly, `buffer.freeze` / `buffer.isfrozen` might take up a fastcall slot in the VM.

## Alternatives

- Do Nothing. It will not be possible to create "immutable" / "sandboxed" / "readonly" buffers due to the implications
on performance and design complexity.
