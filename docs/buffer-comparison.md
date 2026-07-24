# Buffer Data Comparison

## Summary

This RFC proposes extending the Luau buffer library to support comparing the byte sequences of two buffers. The primary API is `buffer.compare(b1, b2)`, which performs a three-way lexicographic comparison (similar to C `memcmp`) and returns an integer: `0` if the sequences are equal, a negative value if `b1` is less than `b2`, and a positive value if `b1` is greater.

## Motivation

Luau currently lacks a built-in way to compare two buffers. Developers who need equality checks must convert buffers to strings and compare those strings, which adds GC pressure and is a poor fit for performance-critical code.

Buffers also cannot be ordered with `<` or `==` in the same way as strings. Anything that needs “less than” or “greater than” on raw bytes—`table.sort` with a comparator, binary search on a sorted list of keys, or custom ordered structures—needs a three-way result. Without a builtin, the choices are: convert to string and use string ordering (allocation + GC), or write a Luau loop that reads bytes with `buffer.readu8` and returns -1/0/1. That loop runs in *your* code (often inside a sort comparator, so it may run many times per comparison). `buffer.compare` moves that work into the VM as one call over the backing memory, same idea as C `memcmp`.

## Design

### `buffer.compare(b1: buffer, b2: buffer) -> number`

**Semantics**:

1. Validate types: both arguments must be buffers; otherwise throw an error.
2. Compare the two byte sequences in unsigned lexicographic order from offset `0`, like `memcmp` on the shared prefix: find the first index where the bytes differ; if `b1`’s byte is smaller, return a negative integer, if larger, return a positive integer (non-zero magnitude is implementation-defined; only the sign is guaranteed).
3. If the bytes match for `min(len(b1), len(b2))` but the lengths differ, the shorter buffer is less (negative if `b1` is shorter, positive if `b2` is shorter).
4. If lengths and every byte match, return `0`.

Equality checks use `buffer.compare(b1, b2) == 0`.

### Why this is better

Instead of converting buffers to strings, a single call compares bytes in place. That avoids string allocations and supports both equality and ordering in one primitive.

### Example 1 – Simple comparison

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

const equal = buffer.compare(my_buf1, my_buf2) == 0
```

### Example 2 – DeepCompare

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
    return buffer.compare(v1, v2) == 0
end
```

### Example 3 – Sorting

```luau
table.sort(entries, function(a, b)
    return buffer.compare(a.key, b.key) < 0
end)
```

## Drawbacks

- Library footprint: adds another function to the buffer namespace.
- Strict whole-buffer semantics: compares entire sequences only. Comparing sub-regions without optional offset/count parameters requires slicing or separate buffers (see Alternatives).
- Slightly less direct for equality-only call sites than a dedicated boolean API (`buffer.compare(a, b) == 0` vs `buffer.eq(a, b)`).

## Alternatives

### Boolean equality (`buffer.eq`)

Provide `buffer.eq(b1, b2) -> boolean` that returns `true` only when both buffers have the same length and every byte matches; otherwise `false`.

Design: `buffer.eq(b1: buffer, b2: buffer) -> boolean` — validate types, compare length (early `false` if unequal), then compare bytes and return `false` on first mismatch or `true` if the loop completes.

Pros:

- More intuitive when the only question is “are these equal?”
- Avoids interpreting a numeric comparison result.

Cons:

- Does not support sorting, ordered structures, or three-way branches without a separate API or manual loops.
- Two comparison entry points if both equality and ordering are needed in the same codebase.

### Ranged comparison support

Extend the signature with optional offsets and counts, matching patterns such as `buffer.copy`:

Design: `buffer.compare(b1, b2, [b1_offset], [b2_offset], [count])` (or the same optional parameters on `buffer.eq`).

Pros:

- Compare windows inside larger buffers without allocating sub-buffers.

Cons:

- More argument parsing and documentation surface at the C/Luau boundary for a less common case.

### Status quo

Developers continue using `buffer.tostring` and string comparison.

Impact: short-lived GC pressure in high-throughput network or serialization paths; performance traded for simplicity.
