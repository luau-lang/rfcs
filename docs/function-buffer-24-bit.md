# 24-Bit Operations for Buffers

## Summary

The built-in `buffer` library will gain 4 new functions:

- `buffer.writeu24`
- `buffer.readu24`
- `buffer.writei24`
- `buffer.readi24`

These functions will read and write 24-bit integers to and from a buffer.

## Motivation

When working with or creating binary data, it's often desirable to reduce the amount of data written by as much as possible. In the event a 32-bit integer is excessive and a 16-bit integer is too small, people often desire 24-bit integers because it is the next byte-aligned integer width.

Implementing `writeu24` and `readu24` using the existing `buffer` functions is trivial, but it is inefficient. It requires two seperate bounds checks for the `buffer` function calls, and requires some form of bit manipulation, whether it be through `bit32` or arithmetic. Implementing `readi24` and `writei24` may also be difficult for some people, as it requires considering sign extensions and two's complement representations. This means that implementations tend to be error prone unless the implementor knows what they are doing.

Both of these issues can be overcome by simply providing an efficient and correct implementation for handling 24-bit integers.

## Design

Reading and writing both signed and unsigned integers is well defined by the [the buffer RFC](./type-byte-buffer.md). This RFC simply extends the widths supported to include 24-bit integers.

For the sake of clarity, the functions would have the following type signatures:

`buffer.writeu24(b: buffer, offset: number, value: number): ()`

`buffer.writei24(b: buffer, offset: number, value: number): ()`

`buffer.readu24(b: buffer, offset: number): number`

`buffer.readi24(b: buffer, offset: number): number`

This matches the signatures of comparable functions in the `buffer` library.

## Drawbacks

There is a general expectation that `buffer` functions be given a built-in optimization and that they lower to efficient instructions when native codegen is enabled. This means that any additions like this will either break that trend or take up valuable built-in slots and add to the overall complexity of native codegen.

Adding support functions in this nature may raise the logical question "why not add X?" because it is accomodating a common use case while neglecting others. This is considered a relatively minor drawback as anyone is allowed to submit an RFC if they wish.

## Alternatives

We could simply not do this. It would be fast, easy, and free. The consequence is that everyone who needs 24-bit integer support for buffers must implement it themselves, potentially introducing bugs.

An interesting alternative is a `buffer` function to read/write arbitrary width integers. This solves the problem for 24-bit integers but it does not lend itself to having an efficient implementation. Supporting potentially non-byte aligned integer widths (e.g. 1 bit) has complicated semantics, and implementing support for it efficiently would be very difficult. Thus, this design was rejected.