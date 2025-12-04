# Extend buffer with a metatable

## Summary

Extend `buffer.create` with a second argument which provides a metatable for that buffer.
The metatable and some of its keys will be frozen for potential improvements in access.

## Motivation

Providing a buffer object with a metatable will allow creation of objects with small storage footprint which can also be matched to native structures on the side of the application using Luau. 

By having a metatable, these buffer objects can have properties, methods and other kinds of operations.

This will also allow the behavior to be defined and extended on the Luau side, compared to userdata that is strictly defined by the host.

In a way, this will provide an alternative to `ctypes` in luajit.

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

Or any other custom way the developer wants property access to be performed.

## Design

`buffer.create(size: number, metatable: table): buffer`

Buffer construction will accept a second argument which will be a metatable for the buffer object.

This metatable will be frozen.
In addition to that, `__index` and `__newindex` fields will also be frozen if they are a table.
Table freezing will have the same limitation as `table.freeze`, throwing an error if table metatable is locked.
Any of these tables can be frozen before the call.

When `__index` or `__newindex` is a function, VM will be allowed to ignore changes to the environment of those function.

The freezing is performed to provide additional guarantees for field access and other metamethod evaluation to unlock potential optimization opportunities.
This is similar to limitations placed on ctypes by luajit.
Having said that, this RFC doesn't make a promise of a particular implementation making those optimizations, so should be viewed as a buffer object usability improvement first.

VM metatable lookups and `getmetatable` will look for the buffer object metatable first and then fall back to the global buffer metatable.
This preserves the existing buffer extensibility point for hosts.

`setmetatable` will not be supported on buffer objects, the metatable reference is immutable.

Equality checks in the VM will call `__eq` for buffers similar to tables and userdata.

`__type` metatable key is ignored by `typeof`. As before, only host is allowed to define type names.

In order for the typechecker to understand buffers with attached metatables, we propose extending the intersections to be allowed on buffers, similar to `extern` types:

`type Obj = buffer & { x: number, y: number, z: number }`

## Drawbacks

This increases buffer size by 8 bytes, with the bigger impact on 0-8 byte buffers going from 16 to 24 bytes and allocated from 32 byte page.

This RFC also introduces a special evaluation rule for metamethod functions.
While it is introduced for potential improvement in caching of operations, this might come at a surprise to users of deprecated environment modification functions.

## Alternatives

Instead of extending the buffer object with a limited set of functionality, we might pursue a new kind of object like Records were which can build internal field mappings for an easier optimization path.

Another possibility is some alternative way of specifying the fields that would support building both the right __index function and internal acceleration structures.
