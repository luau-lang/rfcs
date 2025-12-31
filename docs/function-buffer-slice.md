# `buffer.slice`

## Summary

Add a new function (`buffer.slice(buf, offset, size)`) which creates buffer slices without allocating new memory.

## Motivation

This RFC aims to increase safety and ease-of-use; there are many situations where it is impractical or impossible to
provide safety to an entire buffer while sending a portion of that memory to a consumer. For example, trimming a buffer
to be used in a runtime application, or creating a virtual heap. All available solutions are *very* incomplete, and
often require excess allocations/copies. To summarize, currently there is no "good" way to represent a mutable portion of an
existing buffer.

## Design

### API

The `buffer` library will gain a new function, `buffer.slice`, which will create a buffer object from a buffer, offset,
and size. This new buffer holds a reference to its root, and behaves homogeneously:

```luau
local buf = buffer.create(16)
local slice = buffer.slice(buf, 8, 8)
buffer.writeu32(slice, 0, 1234)
print(buffer.readu32(slice, 0)) -- 1234
print(buffer.readu32(buf, 8)) -- 1234
```

Any slice which begins at an offset inside the source buffer and ends inside the buffer is an allowed slice. This
includes slices starting at offset 0 with size equal to the parent buffer, as well as empty slices. Slices should behave
homogeneously with buffers, because they **are** buffers. This means slices can, themselves, be sliced.

## Drawbacks

- Garbage collection for buffers is complicated slightly by the existence of slices. Buffers are largely designed around
performance, so this is an important consideration.
- This increases the number of standard library functions, which naturally makes the language harder to "learn". More
importantly, `buffer.slice` will probably take up a fastcall slot in the VM. Considering the motivation(s) for slices,
this drawback is ostensibly "worth it".

## Alternatives

- Do Nothing. This forces users to allocate repetitive memory in order to trim a buffer or create a virtual heap.
Generally, it is useful to pass around a segment of a mutable whole.
- Structure arguments as `buffer.slice(buf, start, finish)` rather than offset and size. There is no particular reason
this would be worse, but the design proposed in this RFC is more consistent with existing functions (`buffer.copy`)
