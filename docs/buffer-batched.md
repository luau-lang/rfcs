# Batched Buffer Read/Write Extension

## Summary

This RFC proposes extending the Luau `buffer` library to support batched byte-based read/write operations. By introducing functions `buffer.unpack*` which returns `...number` in a single call, and `buffer.pack*` which writes `...number` in a single call. This reduces verbosity, provides performance that scales over conventional `buffer.read*`/`buffer.write*` & boilerplate via minimized Luau/C boundary crossing, reasons well under forward-compatible types that interop with `number`, and enables elegant type conversion patterns while maintaining the buffer's raw, unopinionated nature.

## Motivation

Working closely with buffers often requires verbose loops or repeated function calls, particularly when populating vectors or other constructors that accept `...number`. Each individual call crosses the Luau/C boundary, which incurs significant overhead and limits performance gains in interpreted contexts. Batched operations would drastically reduce these crossings, offering a low-hanging fruit for major performance improvements across the language. Additionally, batched writes enable concise type truncation and conversion (e.g., f64 to u16) without manual bit manipulation or intermediate variables. This RFC builds on prior concepts but focuses strictly on `buffer` and `number` types to minimize the implementation surface and maximize ergonomic benefits for existing Luau APIs and user-provided functions that accept variadic numbers.

## Design

### Byte-Based Batched Reads
`buffer.unpack*` would function similarly to `buffer.read*`, gaining a third `count: number` parameter, accepting a minimum `number` value of 1, guaranteeing an unambiguous return type. The type suffix indicates the byte size per value (u8/i8=1, u16/i16=2, u32/i32/f32=4, f64=8). The index would step internally after each read, with the number of steps provided by `count - 1`, and the number of bytes advanced per stepped inferred from the function's type suffix. Going OOB would error the same way as `buffer.read*`, preventing any values from being returned.

- **Current:** `buffer.read*(buffer: buffer, index: number) -> number`
- **Proposed:** `buffer.unpack*(buffer: buffer, index: number, count: number) -> ...number`

Batched reads allow direct forwarding of return values into constructors or any Luau function accepting `...number`.

Example: Consolidating the need for type-specific buffer read/writes originating from [the luau team's RFC](https://github.com/luau-lang/rfcs/pull/198):

*Current Luau capabilities:*
```luau
local VECTOR_NDIM = pcall(function() return (vector.one :: any).w end) and 4 or 3
local VECTOR_SIZEOF = VECTOR_NDIM * 4

local function buf_ReadVec3(buf: buffer, i: number)
  return
    buffer.readf32(buf, i),
    buffer.readf32(buf, i + 4),
    buffer.readf32(buf, i + 8)
end

local function buf_ReadVec4(buf: buffer, i: number)
  return
    buffer.readf32(buf, i),
    buffer.readf32(buf, i + 4),
    buffer.readf32(buf, i + 8),
    buffer.readf32(buf, i + 12)
end

local buf_ReadVec = if VECTOR_NDIM == 4 then buf_ReadVec4 else buf_ReadVec3

local function doThing(buf: buffer)
  for i = 0, buffer.len(buf) - 1, VECTOR_SIZEOF do
    local v = (vector.create :: any)(buf_ReadVec(buf, i))
    -- ...
  end
end

doThing(buffer.create(128 * VECTOR_SIZEOF))
```

*Proposed batch approach:*
```luau
local VECTOR_NDIM = pcall(function() return (vector.one :: any).w end) and 4 or 3
local VECTOR_SIZEOF = VECTOR_NDIM * 4

-- No 'buf_ReadVec' function needed; less logic to be considered, removing complexity from interacting with buffers in this case.

local function doThing(buf: buffer)
  for i = 0, buffer.len(buf) - 1, VECTOR_SIZEOF do
    local v = (vector.create :: any)(buffer.unpackf32(buf, i, VECTOR_NDIM))
    -- ...
  end
end

doThing(buffer.create(128 * VECTOR_SIZEOF))
```

### Byte-Based Batched Writes
`buffer.pack*` would function similarly to `buffer.write*`, where `value: number` would instead accept `...number`, allowing for multiple numbers to be written in 1 call. The type suffix indicates the byte size per value (u8/i8=1, u16/i16=2, u32/i32/f32=4, f64=8). The index would step internally after each write, with the number of steps provided by `count - 1`, and the number of bytes advanced per stepped inferred from the function's type suffix. OOB would be checked before writing to the buffer and runtime error if bounds would be exceeded, preventing accidental partial writes.

- **Current:** `buffer.write*(buffer: buffer, index: number, value: number) -> ()`
- **Proposed:** `buffer.pack*(buffer: buffer, index: number, ...number) -> ()`

**Example: Convert buffer u32 numbers to buffer u8 numbers, with versatility on buffer choice and index**

*Current Luau capabilities (`buffer.readu32`, `buffer.writeu8`):*
```luau
local function u32_to_u8(from_b: buffer, to_b: buffer, from_i: number, to_i: number, from_n: number)
  local to_cursor = to_i
  local from_cursor = from_i
  for i = 1, math.max(from_n or 1, 1) do
    buffer.writeu8(to_b, to_cursor, buffer.readu32(from_b, from_cursor))
    to_cursor += 1 -- to_buf is u8, need to increment by 1
    from_cursor += 4 -- from_buf is u32, need to increment by 4
  end
end
```

*Proposed batched approach (`buffer.unpacku32`, `buffer.packu8`):*
```luau
local function u32_to_u8(from_b: buffer, to_b: buffer, from_i: number, to_i: number, from_n: number)
  buffer.packu8(to_buf, to_i, buffer.unpacku32(from_buf, from_i, from_n))
end
```

**Example: Write vector to buffer**

*Current Luau capabilities (`buffer.writef32`):*
```luau
local VECTOR_NDIM = pcall(function() return (vector.one :: any).w end) and 4 or 3
local VECTOR_SIZEOF = VECTOR_NDIM * 4

local function buf_WriteVec3(buf: buffer, i: number, vX: number, vY: number, vZ: number)
  buffer.writef32(buf, i, vX)
  buffer.writef32(buf, i + 4, vY)
  buffer.writef32(buf, i + 8, vZ)
end

local function buf_WriteVec4(buf: buffer, i: number, vX: number, vY: number, vZ: number, vW: number)
  buffer.writef32(buf, i, vX)
  buffer.writef32(buf, i + 4, vY)
  buffer.writef32(buf, i + 8, vZ)
  buffer.writef32(buf, i + 12, vW)
end

local buf_WriteVec = if VECTOR_NDIM == 4 then buf_WriteVec4 else buf_WriteVec3
local b = buffer.create(VECTOR_SIZEOF)
local v = vector.one
buf_WriteVec(b, 0, v.x, v.y, v.z, VECTOR_NDIM == 4 and (v :: any).w)
```

*Proposed batch approach (`buffer.packf32`):*
```luau
local VECTOR_NDIM = pcall(function() return (vector.one :: any).w end) and 4 or 3
local VECTOR_SIZEOF = VECTOR_NDIM * 4

-- No 'buf_WriteVec' function needed; less logic to be considered, removing complexity from interacting with buffers in this case.

local b = buffer.create(VECTOR_SIZEOF)
local v = vector.one
buffer.packf32(b, 0, v.x, v.y, v.z, VECTOR_NDIM == 4 and (v :: any).w)
```

### Emergent Patterns
Combining extended read/write behaviors enables seamless pipelines, such as reading `...number` inputs as f64 and immediately translating them to the valid type suffix before writing to the buffer at its given size. Small working buffers may result in being created just to take advantage of this behaviour without requiring more-verbose intermediate variables or loops.

## Drawbacks

- **Out-of-Bounds Risks:** Incrementing the index per stepped read/write in a single call would allow for OOB runtime errors in cases where the user provides more data than the buffer is appropriately sized for. Under the initial type implementation and latter [stagnation of a prior RFC](https://github.com/luau-lang/rfcs/pull/155), the `buffer` is wholly seen as raw data without opinion. The drawback of this drawback is that it sounds like a motive.
- **Initial Fastcall Cost:** Adding dedicated `pack*`/`unpack*` buffer library functions would cost fastcall allocations. Forward-compatibility based on future type APIs being interoperable with `number` would need to be considered to warrant this. More-immediate mitigations are explored in the alternatives section.

## Alternatives

- **Reduced number type suffix support:** The argument for this RFC is strongest regarding types f32 (`vector` components), u32/i32 (resultant `bit32` usage on `number`), and f64 (`number`). Implementing u8/i8/u16/i16 in scenarios where bitpacking multiple u8/i8/u16/i16's into a u32/i32 may be redundant. u32/i32/f32/f64 implementation would be an acceptable MVP.
- **Status Quo:** Developers continue writing verbose loops or chaining single-value calls, accepting higher fastcall overhead, reduced ergonomics, and slower interpreted execution.
- **Incremental Type-Specific RFCs:** Separate RFCs for improved type-to-buffer will continue to be created, some having a stronger argument for use than others, with each implementation accumulating towards exceeding what the fastcall allocation would be from implementing this RFC. Maintenance burden would be higher, allowing for more foot-in-the-door scenarios for discourse on RFCs based on type-to-buffer interoperability.
- **Manual Bit Manipulation:** Continue relying on `bit32` and manual loops for type conversion and bitpacking, which is more verbose, error-prone, and less performant than native batched buffer operations.
