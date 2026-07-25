# Buffer Data Comparison

## Summary

This RFC proposes extending the Luau buffer library to compare byte sequences in two buffers, including optional sub-regions. The primary API is `buffer.compare(b1, b1_offset, b2, b2_offset?, count?)`, which performs a three-way lexicographic comparison (like C `memcmp` on a chosen window) and returns an integer: `0` if the compared regions are equal, a negative value if the `b1` region is less than the `b2` region, and a positive value if it is greater.

## Motivation

Luau currently lacks a built-in way to compare two buffers. Developers who need equality checks must convert buffers to strings and compare those strings, which adds GC pressure and is a poor fit for performance-critical code.

Buffers also cannot be ordered with `<` or `==` in the same way as strings. Anything that needs “less than” or “greater than” on raw bytes—`table.sort` with a comparator, binary search on a sorted list of keys, or custom ordered structures—needs a three-way result. Without a builtin, the choices are: convert to string and use string ordering (allocation + GC), or write a Luau loop that reads bytes with `buffer.readu8` and returns -1/0/1. That loop runs in *your* code (often inside a sort comparator, so it may run many times per comparison). `buffer.compare` moves that work into the VM as one call over the backing memory, same idea as C `memcmp`.

Comparing only part of a buffer (a header, checksum field, or record inside a larger arena) is common in protocols and serialization. Without ranged comparison, developers allocate a sub-buffer or copy into a temporary buffer before comparing—reintroducing the allocation cost the buffer type is meant to avoid.

## Design

### `buffer.compare(b1: buffer, b1_offset: number, b2: buffer, b2_offset: number?, count: number?) -> number`

Signature follows the same offset/count pattern as `buffer.copy`: a required start offset on the first buffer, optional start on the second, optional length.

**Defaults** (same spirit as `buffer.copy`):

- If `b2_offset` is `nil` or omitted, it defaults to `0`.
- If `count` is `nil` or omitted, compare from the given offsets through the end of each buffer: effectively `count = min(len(b1) - b1_offset, len(b2) - b2_offset)` (if either remaining length is negative, behavior matches other buffer functions: error on invalid offset).

**Bounds**: If `count` is provided, both regions `[b1_offset, b1_offset + count)` and `[b2_offset, b2_offset + count)` must lie within their buffers; otherwise an error is thrown, consistent with `buffer.copy` and read/write helpers.

**Semantics**:

1. Validate types: `b1` and `b2` must be buffers; offsets and `count` must be numbers where applicable.
2. Determine `n`, the number of bytes to compare: explicit `count` if given, otherwise the minimum remaining length from each offset as above.
3. Compare `n` bytes at `b1_offset` and `b2_offset` in unsigned lexicographic order: on the first differing byte, return negative if `b1`’s byte is smaller, positive if larger (non-zero magnitude is implementation-defined; only the sign is guaranteed).
4. If all `n` bytes match and `count` was **explicit**, return `0` (extra bytes beyond the window are ignored).
5. If all `n` bytes match and `count` was **omitted** but the remaining lengths after the offsets differ, the shorter remaining region is less (negative if `b1`’s tail is shorter, positive if `b2`’s tail is shorter).
6. If all compared bytes match and the compared spans are the same length, return `0`.

Equality for two full buffers from the start: `buffer.compare(b1, 0, b2, 0) == 0` or `buffer.compare(b1, 0, b2) == 0`.

### Example 1 – Full-buffer equality

Manual `buffer.tostring` and string comparison:

```luau
const my_buf1 = buffer.create(100)
const my_buf2 = buffer.create(100)
-- ... (write data to them)

const str_buf1 = buffer.tostring(my_buf1)
const str_buf2 = buffer.tostring(my_buf2)

const equal = str_buf1 == str_buf2
```

Using `buffer.compare`:

```luau
const my_buf1 = buffer.create(100)
const my_buf2 = buffer.create(100)
-- ... (write data to them)

const equal = buffer.compare(my_buf1, 0, my_buf2, 0) == 0
```

### Example 2 – Compare a header inside a packet

```luau
const HEADER_LEN = 16
const pkt_a = buffer.create(256)
const pkt_b = buffer.create(256)
-- ... payload may differ; only compare header

const same_header = buffer.compare(pkt_a, 0, pkt_b, 0, HEADER_LEN) == 0
```

### Example 3 – DeepCompare

```luau
const simpleEq = {
    number = true,
    string = true,
    boolean = true,
    userdata = true, -- for this example
}

const function DeepCompare(v1, v2)
    const t1, t2 = type(v1), type(v2)
    if t1 ~= t2 then
        return false
    end

    if simpleEq[t1] then
        return v1 == v2
    end

    if t1 == "table" then
        for k, v in v1 do
            if not DeepCompare(v, v2[k]) then
                return false
            end
        end
        return true
    end

    -- buffer case
    return buffer.compare(v1, 0, v2, 0) == 0
end
```

### Example 4 – Sorting

```luau
table.sort(entries, function(a, b)
    return buffer.compare(a.key, 0, b.key, 0) < 0
end)
```

## Drawbacks

- Library footprint: adds another function to the buffer namespace.
- More arguments and defaulting rules than a two-buffer-only API; slightly more work at the C/Luau boundary and in documentation.
- Slightly less direct for equality-only call sites than a dedicated boolean API (`buffer.compare(...) == 0` vs `buffer.eq(...)`).

## Alternatives

### Whole-buffer comparison only

`buffer.compare(b1: buffer, b2: buffer) -> number` with no offset or `count` parameters—always compares from offset `0` through `min(len(b1), len(b2))`, then applies length ordering if the shared prefix matches.

Pros:

- Smaller API surface and simpler binding code.
- Natural call sites when buffers are always compared in full (`buffer.compare(a, b)`).

Cons:

- Sub-regions require `buffer.tostring` on a slice, a manual `readu8` loop, or copying into a temporary buffer—allocation or extra script cost.
- Duplicates logic if ranged comparison is added later as a second function.

### Boolean equality (`buffer.eq`)

Provide `buffer.eq(b1, b2, [offsets/count]?) -> boolean` that returns `true` only when the compared regions have the same length and every byte matches; otherwise `false`.

Pros:

- More intuitive when the only question is “are these equal?”
- Avoids interpreting a numeric comparison result.

Cons:

- Does not support sorting, ordered structures, or three-way branches without a separate API or manual loops.
- Two comparison entry points if both equality and ordering are needed in the same codebase.

### Status quo

Developers continue using `buffer.tostring` and string comparison.

Impact: short-lived GC pressure in high-throughput network or serialization paths; performance traded for simplicity.
