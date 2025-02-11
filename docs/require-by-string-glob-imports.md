# Glob import syntax for require-by-string

## Motivation

- Cross-runtime automated testing
- Auto-requiring ecs systems
- Commands for things like discord bots
- Module loaders in general
- Type safety (even if it's typed to be unknown, it's better than typecasting to any.)
- Cyclical dependency reporting
- This would be required for static cross-compilation or whatever the term is

## Design

- directory/\*\* for descendants
- directory/\* for children
- Alphanumeric ordering

```lua
local children = require("directory/*)") -- array
local descendants = require("directory/**") -- array
```

## Alternatives

- Provide a dictionary where { [name]: return value }?
  - In descendants, this might cause duplicates.
  - Include file extension? Can have duplicates if no file extension.

## Drawbacks

- Cross-runtime ordering?
- OS causes implementation concerns?
- Type concerns?
- Might promote bad patterns?
