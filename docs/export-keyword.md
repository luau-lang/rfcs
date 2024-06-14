# Export Keyword

## Summary

Allow users to export methods and values from their libraries through the use of the export keyword.

## Motivation

Users can export types from their libraries through the use of the export keyword.

```luau
export type Point = { x: number, y: number }
```

Consumers of these libraries are able to access exported types by indexing the libraries identifier.

```luau
local Library = require("Library")
local point : Library.Point = { x = 0, y = 0 }
```

Right now, this is the only use of the export keyword but it would be great if we let users export more with it to offer more utility and make exporting library methods and values easier.

## Design

### Syntax
You can prefix any non-local declaration with the export keyword in the top level scope of a module. For values this looks like:

```luau
export version = "5.1"
```

While methods use:

```luau
export function init()
  -- do a thing
end
```

### Behavior

When a value or method is prefixed with export, it is automatically added to a hidden export table which is frozen as soon as the module returns.

Take the following module:

```luau
export type Point = { x: number, y: number }

export function distance(a: Point, b: Point)
  local x, y = a.X - b.X, a.Y - b.Y
  return math.sqrt(x * x + y * y)
end
```

This essentially becomes sugar for:

```luau
local _EXT = {}

type Point = { x: number, y: number }
type _EXT.Point = Point -- note: this doesn't actually work today but one day it should

local function distance(a: Point, b: Point)
  local x, y = a.X - b.X, a.Y - b.Y
  return math.sqrt(x * x + y * y)
end
_EXT.distance = distance

return table.freeze(_EXT)
```

If the user attempts to assign to an exported identifier then we would throw an error explaining that the interface cannot be changed once it has been exported.

### Nuances
Due to the implementation, most things should "just-work". Here are some examples to consider:

#### Calling an Exported Function
You can call an exported function as it's registered as a local before being added to the export table.

```luau
export function distance(a: Point, b: Point)
  local x, y = a.X - b.X, a.Y - b.Y
  return math.sqrt(x * x + y * y)
end

distance({0, 0}, {1, 1})
```

#### Nested Tables
You can export tables with additional values inside of them.

```luau
export triangle = {}

function triangle.draw()

end
```

Something important to note here is that the nested table, `triangle` is not frozen.

#### Returns
Today, you can use the export keyword along with a return statement at the end of your module. If you use the export keyword with a value however we will throw an error if you also attempt to return.

```luau
export function distance(a: Point, b: Point)
  local x, y = a.X - b.X, a.Y - b.Y
  return math.sqrt(x * x + y * y)
end

return table.freeze {
  distance = distance
}
```

## Drawbacks
The keyword already exists for types so there's not much cost in us adding support to values. The main drawback is that it's another way to do module exports. We already have a way to do that, do we really need another?

## Alternatives

### Do Nothing
It's already possible to export values from modules using a return statement. We don't actually need to do this, it's more of a nice-to-have.

### Automatically Export Globals
Luau has a separate global scope for each module rather than a shared global scope across the entire program. We could automatically export any values that are stored in the global scope. This isn't backwards compatible though, we'd likely need a new import mechanism to resolve this.