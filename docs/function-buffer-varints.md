# Varint Functions for Buffers

## Summary

This RFC proposes the addition of new functions to the buffer API:

- `buffer.readleb128` 
- `buffer.readleb128`
- `buffer.readuleb128`
- `buffer.writeuleb128` 

These functions will write and read (U)LEB-128 variable-width integers (varints) from a buffer.

## Motivation

One of the top use cases for buffers is compressing and serializing data as small as possible. Numbers consume 8 bytes in their raw form but can be serialized in smaller sizes using buffers to reduce size. However, writing code to serialize as the smallest size can be tedious and confusing for less-experienced programmers.  

The following are implementations of ULEB-128 reading/writing in Luau:

```lua
local function writeuleb128(stream, value)
  while value >= 0x80 do
    buffer.writeu8(bit32.bor(bit32.band(value, 0x7f), 0x80))
    value = bit32.rshift(value, 7)
  end
  buffer.writeu8(value)
end
```

```lua
local function readuleb128(stream)
  local result = 0
  local bits = 0
  while true do
    local byte = buffer.readu8(stream)
    result = bit32.bor(result, bit32.lshift(bit32.band(byte, 0x7f), bits))
    if byte < 0x80 then
      break
    end
    bits = bits + 7
  end
  return result
end
```

The functions above are inefficient and difficult to understand compared to a native implementation. Implementations will also be needed for the corresponding signed functions.

## Design

The `buffer` library will receive 4 new functions:

```
buffer.readleb128(b: buffer, offset: number): number
buffer.readuleb128(b: buffer, offset: number): number

buffer.writeleb128(b: buffer, offset: number, value: number): ()
buffer.writeuleb128(b: buffer, offset: number, value: number): ()
```

Since other numbers in the buffer library have unsigned and signed implementations, it also makes sense to include both options for varints.

## Drawbacks

The only drawback known is a marginal increase in built-in complexity. However, the performance benefit from having native implementations of these functions outweighs the negligible change in complexity and is not a serious concern.

## Alternatives

Serialization and deserialization for varints can be recreated directly in Luau. However, the algorithm for doing this may be complicated for less-experienced programmers as it involves bitwise operations. Additionally, the algorithm requires repeated buffer reads and calls to bitwise functions to function correctly, which is far less performant than it could be in native code.

It is also possible to have a function that serializes a number in the smallest amount of bits it can fit into. However, to read it, the amount of bits it was serialized in would also have to be included. This count of how many bytes to read would also have to be stored in the buffer, which adds an extra byte of unnecessary data.
