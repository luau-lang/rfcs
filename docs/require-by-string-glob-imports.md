# Glob import syntax for require-by-string

## Motivation

- Many patterns utilize current dynamic requires, or load in luau files manually:
  - ECS systems
  - The "service" pattern
  - Module initializers
  - Tests
- The alternative (writing out manual requires) is quite cumbersome and not feasible to expect developers to do
- Cross-runtime/platform automated testing is currently impossible
- Current dynamic requires are not type safe and can have cyclical dependency errors which are not picked up on. This isn't a conceptual limitation, it is totally possible to report.
- Cross-module compilation may be wanted in the eventual future. But this pattern is common and will not be going away. There should be a way to express "requiring all children of a directory" while still having full cross-module compilation.

## Design

- directory/\*\* for descendants
- directory/\* for children
- Alphanumeric ordering off file name
  - file1
  - file12
  - file13
  - file2
  - file31
  - file3
- "File name" does not include extension!

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
