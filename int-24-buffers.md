# Add 24-bit Integer and Float Buffer Accessors

## Summary

This proposal adds support for 24-bit integer types (`u24`, `i24`) and optionally a 24-bit float-like format (`f24`) to the `buffer` library, enabling more memory-efficient serialization of common data such as RGB colors, compact indices, and domain-specific packed values.

---

## Motivation

A significant number of serialization use cases do not require full 32-bit storage:

* RGB color channels (3 × u8 = 24 bits total)
* small indices / IDs (< 16 million range)
* compact network packets
* ECS component IDs
* voxel/grid data
* domain-specific packed structs

Currently, these are typically stored using:

* `u16` + `u8` write
* manual bit packing via `bit32`
* or multi-field encoding

RGB is naturally a 24-bit packed value (3×u8), currently requiring manual bit packing or multiple writes.
Example (RGB packing today):

```luau
local packed =
    bit32.bor(
        r,
        bit32.lshift(g, 8),
        bit32.lshift(b, 16)
    )
```

Using `u32` for 24-bit data forces either:
- wasted 1 byte per element, or
- misaligned stream packing with manual offset management + possible out of bounds error

This is functional, but:

* requires manual bit manipulation
* introduces boilerplate
* uses unnecessary 2 operations

---

## Design

Introduce the following buffer APIs:

### Unsigned 24-bit integer

```luau
buffer.writeu24(buf, offset, value)
buffer.readu24(buf, offset)
```

* Range: `0 .. 2^24 - 1`
* Storage: 3 bytes
* Endianness: little-endian (consistent with existing buffer behavior)

---

### Signed 24-bit integer

```luau
buffer.writei24(buf, offset, value)
buffer.readi24(buf, offset)
```

* Range: `-2^23 .. 2^23 - 1`
* Two’s complement encoding
* Storage: 3 bytes

---

## Benefits

* Reduces memory usage by 25% for common packed data vs `u32`
* Removes need for manual bit manipulation in common cases
* Improves readability for domain-specific serialization
* Better alignment with real-world data formats (RGB, indices)

---

## Drawbacks

* Increases buffer API surface area
* Adds complexity to serialization layer
* `f24` introduces non-standard floating-point behavior risks
* Slight increase in maintenance burden for VM/runtime buffer code

---

## Alternatives

### 1. Keep using `u32` + bit32 packing

Pros:

* already supported
* flexible
  Cons:
* verbose
* error-prone
* wastes 1 byte per value in many cases

---

### 2. User-level packing utilities

Pros:

* no engine changes needed
  Cons:
* repeated boilerplate
* inconsistent implementations
* worse performance potential than native ops

---

### 3. Byte-wise `u8` composition only

Pros:

* maximal control
  Cons:
* extremely verbose
* inefficient for multi-field values
* shifts burden entirely to user code

---

## Conclusion

Adding 24-bit integer buffer support provides a meaningful middle ground between flexibility and efficiency for high-frequency serialization patterns, particularly in graphics, simulation, and networking domains.
This reduces need for user-space bit manipulation in serialization-heavy code paths.
