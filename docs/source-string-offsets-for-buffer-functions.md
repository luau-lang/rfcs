# Source string offsets for buffer functions

## Summary

Extend `buffer.writestring` and `buffer.fromstring` with optional source string offsets, allowing them to operate on a portion of a source string.

## Motivation

This proposal extends the API introduced by the original [Byte buffer type RFC](https://rfcs.luau.org/type-byte-buffer.html).

`buffer.copy` already supports copying data starting at an offset within a source buffer. `buffer.writestring` and `buffer.fromstring` do not provide the same capability.

When only part of a string needs to be copied, the current approach is to first create a new string with `string.sub` before passing it to the buffer library. This performs an additional string allocation and copy. Since Luau strings are immutable, this cannot be avoided in user code.

Adding optional source string offsets allows both functions to copy directly from the original string. This makes them consistent with `buffer.copy` while avoiding unnecessary string allocations.

## Design

This proposal extends `buffer.writestring` and `buffer.fromstring` with optional source string offsets.

The new parameters are appended to the existing function signatures to preserve backwards compatibility. Placing `valueOffset` before `count` would produce a more natural parameter order, but it would change the meaning of existing calls that pass `count`.

`buffer.fromstring(value: string, valueOffset: number?, count: number?): buffer`

Instantiates the object from a string.

If 'valueOffset' is nil or is omitted, it defaults to 0.

If 'count' is nil or is omitted, the size of the buffer is fixed and equals to the length of the string starting from 'valueOffset'.

If an optional 'count' is specified, the size of the buffer is fixed and equals to 'count'. 'count' cannot be larger than the number of bytes remaining in the string after 'valueOffset'.

`buffer.writestring(b: buffer, offset: number, value: string, count: number?, valueOffset: number?): ()`

Used to write data from a string into the buffer at specified offset.

If 'count' is nil or is omitted, all bytes after 'valueOffset' are taken from the string.

If 'valueOffset' is nil or is omitted, it defaults to 0.

If an optional 'count' is specified, only 'count' bytes are taken from the string starting at 'valueOffset'. 

'count' cannot be larger than the number of bytes remaining in the string after 'valueOffset'.

---

Both functions preserve their existing behavior when the new parameters are omitted.

Unless otherwise specified, attempting to read beyond the end of the source string results in an error.

## Drawbacks

This proposal increases the API surface by adding optional parameters to two existing functions.

The meaning of 'count' depends on 'valueOffset', making the APIs slightly more complex than their current forms.

## Alternatives

The current approach is to create a new string before passing it to the buffer library.

```lua
buffer.writestring(
    buffer,
    offset,
    string.sub(source, begin + 1, begin + count)
)
```

or

```lua
buffer.fromstring(
    string.sub(source, begin + 1, begin + count)
)
```

While functional, both approaches perform an additional string allocation and copy.

Another possible solution would be to introduce string or buffer slices. Such a feature could eliminate allocations in a wider range of scenarios, but it would also require new language syntax, runtime semantics, ownership and lifetime rules, and type checker support.

This proposal instead extends two existing APIs with a small, backwards compatible change that addresses a common use case without introducing new language concepts.