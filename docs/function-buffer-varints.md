
# Varint Functions for Buffers

## Summary

This RFC proposes the addition of new functions to the buffer API:

- `buffer.readleb128` 
- `buffer.writeleb128`
- `buffer.readuleb128`
- `buffer.writeuleb128` 

These functions will write and read (U)LEB-128 variable-width integers (varints) from a buffer.

## Motivation

One of the top use cases for buffers is compressing and serializing data as small as possible. Numbers consume 8 bytes in their raw form but can be serialized in smaller sizes using buffers to reduce size. However, writing code to serialize as the smallest size can be tedious and confusing for less-experienced programmers.  

The following are implementations of ULEB-128 reading/writing in Luau:

```lua
local function writeuleb128(stream, offset, value)
  local start = offset
	
  while value >= 0x80 do
    buffer.writeu8(stream, offset, bit32.bor(bit32.band(value, 0x7f), 0x80))
    value = bit32.rshift(value, 7)
    offset = offset + 1
  end
  buffer.writeu8(stream, offset, value)

  return (offset - start) + 1
end
```

```lua
local function readuleb128(stream, offset)
    local result, shift = 0, 0
    local length = buffer.len(stream)
    local start = offset

    repeat
        local byte = buffer.readu8(stream, offset)
        result = bit32.bor(result, bit32.lshift(bit32.band(byte, 0x7f), shift))
        shift, offset = shift + 7, offset + 1
    until byte < 0x80 or length <= offset

    return result, offset - start
end
```

The functions above are inefficient and difficult to understand compared to a library implementation. In some very common examples such as network event compression or data decompression, these functions can be called hundreds or even thousands of times per second. Library implementations would solve all readability/complexity, performance, and compression efficiency problems.

## Design

The `buffer` library will receive 4 new functions:

```
buffer.readleb128(b: buffer, offset: number): (number, number)
buffer.readuleb128(b: buffer, offset: number): (number, number)

buffer.writeleb128(b: buffer, offset: number, value: number): number
buffer.writeuleb128(b: buffer, offset: number, value: number): number
```

Since other numbers in the buffer library have unsigned and signed implementations, it also makes sense to include both options for varints.

The functions take arguments similar to other read/write functions in the buffer library. However, they differ in that they return the amount of bytes that were read/written. In readleb128/readuleb128, this is the second return value. Having these functions return the count is necessary as it must be known in order for users to keep track of the buffer offset.

## Drawbacks

The only drawback known is a marginal increase in library complexity. However, the performance benefit from having library implementations of these functions outweighs the negligible change in complexity and is not a serious concern.

## Alternatives

Serialization and deserialization for varints can be recreated directly in Luau. However, the algorithm for doing this may be complicated for less-experienced programmers as it involves bitwise operations. Additionally, the algorithm requires repeated buffer reads and calls to bitwise functions to function correctly, which is far less performant than a library implementation.

It is also possible to have a function that serializes a number in the smallest amount of bytes it can fit into. However, to read it, the amount of bytes would also have to be included. This size to read would also have to be stored in the buffer, which adds an extra byte of unnecessary data.
