# Buffer and string slices

## Summary

Add first-class `slice` expressions for `buffer` and `string` values.

A `slice` creates a lightweight view into an existing value without copying the underlying data.

Additionally, expose slicing through a `__slice` metamethod to allow user-defined types to implement custom slicing behavior.

## Motivation

Currently, extracting a portion of a buffer or string requires creating a new object and copying data:

```lua
local slice = string.sub(str, start, finish)
```

### or

```lua
local slice = buffer.create(size)
buffer.copy(part, 0, source, offset, size)
```

#### For high-frequency workloads such as:

* network packet serialization;
* binary protocols;
* ECS state synchronization;
* compression/decompression;
* parsing;


these allocations create unnecessary **GC** pressure.

Slices allow users to work with existing memory regions without copying.

## Design

### Syntax

Add a new postfix slice operator:

```lua
value|start, length|
```

#### Examples:

```lua
local my_buffer = buffer.create(10)

buffer.writeu32(my_buffer, 0, 123)

local slice = my_buffer|0, 4|

print(buffer.readu32(slice, 0))
-- 123
```

#### String example:

```lua
local str = "Hello, World!"

print(str|0, 5|) -- Hello
```

### Semantics

#### A slice

* does not copy the underlying data
* keeps the original value alive
* has its own offset and length
* can be passed anywhere the original type is accepted.

#### Example:

```lua
local data = buffer.create(1024)

local header = data|0, 16|
local payload = data|16, 512|
```

Both slices share the same backing storage.

Mutation behavior For buffer:

#### 1st option

Slices are mutable views:

```lua
local view = buffer|0, 4|

buffer.writeu32(view, 0, 10)
```

modifies the original buffer

#### 2nd option

Slices use `copy-on-write`(COW):

```
local view = buffer|0, 4|

buffer.writeu32(view, 0, 10)
```

detaches storage only when modification occurs.

## Metamethod support

### Add

```lua
__slice(start, length)
```

### Example

```lua
local MyBuffer = setmetatable({}, {
    __slice = function(start, end)
        -- ...
    end
})
```

This allows libraries to implement custom `sliceable` types.



## Drawbacks

* Introduces a new operator into Luau syntax.
* May be unfamiliar to new users
* Slice lifetime semantics need to be carefully specified.
* String slices may keep large backing strings alive longer than expected.

## Alternatives

### `buffer.slice`

Use a normal function:

```lua
local slice = buffer.slice(my_buffer, 0, 4)
```

### Advantages:

* follows existing Lua style
* no syntax changes.

### Disadvantages:

* less ergonomic
* harder to use in parsing-heavy code
* does not generalize naturally to strings/user types
