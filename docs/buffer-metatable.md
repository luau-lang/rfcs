# Extend buffer with a metatable

## Summary

Extend `buffer.create` with a second argument which provides a metatable for that buffer.
The metatable and some of its keys will be frozen for potential improvements in access.

## Motivation

Buffer type today has a single metatable provided by the embedder and shared by all buffer instances.

This limits the flexibility as all buffers share the same behavior and those behaviors cannot be defined by user in Luau.
Even for the embedder, defining the methods in Luau is challenging and host language definitions are used instead.

Providing a buffer object with an individual metatable will allow buffer objects to have properties (direct or computed), methods and operators, specific to certain instances.

This will allow the behavior to be defined and extended on the Luau side, compared to userdata that is strictly defined by the embedder.

This extension preserves the simple small core of the Luau language while providing a flexible extension point for developers.

Such buffers can also be made to match the structure of the host data types and to handle FFI cases.

In a way, this will provide an alternative to luajit 'cdata' in Luau:

```luau
-- A float4 in a buffer
local mt = {
  __index = function(b, field)
    if field == "x" then return buffer.readf32(b, 0)
    elseif field == "y" then return buffer.readf32(b, 4)
    elseif field == "z" then return buffer.readf32(b, 8)
    elseif field == "w" then return buffer.readf32(b, 12)
    else error("unknown field") end
  end,
  __newindex = function(b, field, value)
    if field == "x" then buffer.writef32(b, 0, value)
    elseif field == "y" then buffer.writef32(b, 4, value)
    elseif field == "z" then buffer.writef32(b, 8, value)
    elseif field == "w" then buffer.writef32(b, 12, value)
    else error("unknown field") end
  end
}

local buf = buffer.create(16, mt)

buf.x = 2
buf.y = 4

assert(buf.x + buf.y + buf.z == 6)
```

Or alternatively:
```luau
local mt = {
  __index = {
    x = function(b) return buffer.readf32(b, 0) end,
    y = function(b) return buffer.readf32(b, 4) end,
    z = function(b) return buffer.readf32(b, 8) end,
    w = function(b) return buffer.readf32(b, 12) end,
    setx = function(b, value) buffer.writef32(b, 0, value) end,
    sety = function(b, value) buffer.writef32(b, 4, value) end,
    setz = function(b, value) buffer.writef32(b, 8, value) end,
    setw = function(b, value) buffer.writef32(b, 12, value) end,
    magnitude = function(b) return math.sqrt(b.x * b.x + b.y * b.y + b.z * b.z + b.w * b.w) end,
    normalize = function(b) return ... end,
  },
}

local buf = buffer.create(16, mt)

buf:setx(2)
buf:sety(4)
buf:normalize()

local xn = buf:x()
```

Having all the flexibility of a metatable, developer is free to define whatever properties and behaviors they want and interpret their buffer data in a form suitable for them.

## Design

`buffer.create(size: number, metatable: table): buffer`

Buffer construction will accept a second argument which will be a metatable for the buffer object.

This metatable will be frozen.
In addition to that, `__index` and `__newindex` fields will also be frozen if they are a table.
Table freezing will have the same limitation as `table.freeze`, throwing an error if table metatable is locked.
Any of these tables can be frozen before the call.

When `__index` or `__newindex` is a function, VM will be allowed to ignore changes to the environment of those function.
This is similar to function inlining functionality we have where an inlined function will no longer respect its environment.

The freezing is performed to provide additional guarantees for field access and other metamethod evaluation to unlock potential optimization opportunities.
This is similar to limitations placed on 'cdata' by luajit.
Having said that, this RFC doesn't make a promise of a particular implementation making those optimizations, so should be viewed as a buffer object usability improvement first.

Any chained metatables on table fields are not modified.
This provides an option to have a more dynamic behavior, but giving up on potential performance improvements of a chained indexing access.
This matches behavior of 'cdata' in luajit and will provide a familiarity for developers switching over.

VM metatable lookups and `getmetatable` will look for the buffer object metatable first and then fall back to the global buffer metatable.
This preserves the existing buffer extensibility point for hosts.

`setmetatable` will still not be supported on buffer objects, the metatable reference is immutable.

Equality checks in the VM will call `__eq` for buffers similar to tables and userdata.

`__type` metatable key is ignored by `typeof`, just like it does for tables.
As before, only host is allowed to define type names.
This does not change how it behaved with a global host-provided metatable.

Other metamethods (`__add`/`__len`/...) work as expected and do not change from how they work today for buffers with global host-provided metatable.

Buffer structure size increase by the metatable aligns the data to a 16 byte boundary making it possible to store data which requires 16 byte alignment by the embedder.
This will make buffer data match the alignment guarantee of Luau userdata objects.

In order for the typechecker to understand buffers with attached metatables, we propose extending the intersections to be allowed on buffers, similar to `extern` types:

`type Obj = buffer & { x: number, y: number, z: number }`

## Forwards compatibility

A potential evolution of the buffer type might move the data from the buffer memory block to a separate location with a redirection.

Doing that will enable extending the buffer size without a major impact on performance (shrinking is a problem for range check eliminations).
It might also allow buffer views and slices.

In both cases, this change is compatible. The slice will reuse a section of the data but being a buffer it will have its own metatable.

This makes it possible to have a large memory buffer and sub-allocate buffer objects with attached custom behaviors.

## Drawbacks

This increases buffer size by 8 bytes, with the bigger impact on 0-8 byte buffers going from 16 to 24 bytes, with an overhead decreasing with larger sizes as before.
Starting from 64 bytes the increase is often free based on the current Luau allocator design.
In particular, sizes like 64/96/128/256 do not increase in internal allocation class.

This RFC also introduces a special evaluation rule for metamethod functions.
It is introduced for potential improvement in caching of operations, but might come at a surprise to users of deprecated environment modification functions.
While similar to the effects of function inlining Luau performs, it is an extra item to keep in mind.

## Alternatives

Instead of extending the buffer object with a limited set of functionality, we might pursue a new kind of object like Records were which can build internal field mappings for an easier optimization path.

Another possibility is some alternative way of specifying the fields that would support building both the right `__index` function and internal acceleration structures.
