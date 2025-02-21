# buffer.readbits/writebits

**Status**: Implemented

## Summary

Add `buffer.readbits` and `buffer.writebits` to give developers an easy way to work with bit buffers.

## Motivation

With the release of the buffer type, many developers have asked for a way to write integers of custom non-standard widths and just general bit access, based on previous experience with bit buffer libraries.

While bit operations can be implemented manually, it requires careful bit manipulation, especially when partial byte writes are required or when reading bits is performed sequentially across bytes without a rigid 'bitfield' offset definition.

## Design

`buffer` library will gain two new functions, `readbits` and `writebits`:

```
buffer.readbits(b: buffer, bitOffset: number, bitCount: number): number
buffer.writebits(b: buffer, bitOffset: number, bitCount: number, value: number): ()
```

`bitCount` is an integer in range [0, 32]. Error is thrown if number is not in range.

0 bit width is supported only to not error in generalized cases where bit count is dynamic. Reading 0 bits returns 0 and writing has no effect.

Similar to other numerical buffer read/write functions, if `bitOffset` and `bitCount` cause a bit access outside the bounds of the buffer, an error is thrown.

In `writebits`, `value` is treated as an unsigned 32 bit number. Only `bitCount` least significant bits are written.

In `readbits`, return value is unsigned.

---

Operations are always preformed in little-endian byte order and starting from least significant bits.

This means that:

```lua
buffer.readbits(b, 0, 8) == buffer.readu8(b, 0)
buffer.readbits(b, 0, 16) == buffer.readu16(b, 0)
buffer.readbits(b, 0, 32) == buffer.readu32(b, 0)

buffer.writebits(b, 0, 1, 1)
buffer.readi8(b, 0) == 1

buffer.writebits(b, 1, 1, 1)
buffer.readi8(b, 0) == 3
```

> Only a library function implementation is suggested for implementation, the complexity of the operation of a prototype implementation is unlikely to benefit from a fastcall.

## Drawbacks

Because of the bounds checking requirements, implementation needs a loop to access the buffer bytes, which is potentially slower than what developers can do with existing functions when they know that it's safe to read a larger number and extract bits with bit32 functions.

Because the max size of the buffer is 1GB, `bitOffset` cannot be handled as a 32-bit integer number like byte offset in other buffer functions.

## Alternatives

A signed version of `readbits` is not proposed to simplify the design. Writing and reading signed values can be performed with a bias.
