# `table.pretty` and changes to printing tables

## Summary

We will add a function to display a pretty version of a table for debugging purposes. We will use this function as the backbone to changing the default implementation of `print(table)` in CLI contexts.

## Motivation

In Roblox Studio, printing out a table with "Log mode" off will display an interactive table. In roblox-cli, Lune, and luau.exe, it will print the memory address, which is faintly useful, and makes debugging tables a massive chore. Making tables pretty print is a clear win in this regard.

Adding a function to create these pretty forms and using that as the underlying implementation behind `print` is useful for custom loggers that do not use print and go to file, or for something like debug commands.

## Design

### `table.pretty`

The following function will be added to the standard library:

`table.pretty(input: { [any]: any }, indentation: number?): string`

`indentation` defaults to 2.

For a non-empty table, this will perform the following behavior:

1. The initial opening and ending brace will be on their own line
2. Form the string `[$key]: $value,` for every key and value, based on the same behavior as generalized iteration, even using `__iter` if it exists.
	- `$key` and `$value` will be pretty printed if it is both a table and if it would be acceptable to do so (no `__metatable`). This will be indented `indentation` number of spaces.
	- If `$value` is a string, it will be wrapped in quotes.
	- Otherwise, `$key` and `$value` will be the result of `tostring(value)`.
	- The comma is always included.
	- `$key` is not included if it is a number part of a sequence of keys starting with `1`.
3. In the event of a cyclic table with no tostring implementation, the literal string `"<cyclic: memory address>"` will be put in its place.

If indentation is 0, then it will all be put onto a single line, with spaces being used in place of new lines. In this case, trailing commas will not be present.

#### Examples

```lua
print(table.pretty({
	width = 12,
	toppings = {
		"pepperoni",
		"sausage",
	},
}))
```

-->

```
{
  width = 12,
  toppings = {
    "pepperoni",
    "sausage",
  },
}
```

---

```lua
print(table.pretty({
  "my cool",
  "mixed table",

  key = "value",
  [150] = "awesome",
}))
```

-->

```
{
  "my cool",
  "mixed table",
  key = "value",
  [150] = "awesome",
}
```

---

```lua
print(table.pretty({
  "my",
  "table",
  {
    x = 1,
    y = 2,
  }
}, 0))
```

-->

```
{ "my", "table", { x = 1, y = 2 } }
```

---

```lua
local t = { x = 1 }
t.y = t
print(table.pretty(t))
```

-->

```
{
  x = 1,
  y = <cyclic: 0xDEADBEEF>,
}
```

#### Caveats
An empty table will always produce "{}".

Calling `table.pretty` on an input table with a custom `__metatable` will throw an error.

### Changes to printing tables

The default behavior of printing will change from calling `__tostring` to instead using `__tostring` if it exists, and `table.pretty(t)` if it does not. The existing behavior of printing a memory address will still be available through `print(tostring(t))`, which is already useful in a context like Roblox when working with Log mode.

#### Roblox
While Luau is not Roblox specific, it is nevertheless useful to clarify the expected behavior in that environment. Roblox Studio is expected to work the same way as it does now when "Log mode" is disabled--an interactive table expansion. When "Log mode" is enabled, the new printing behavior will take effect.

## Drawbacks
TODO:
- LogService breaking change

## Alternatives
TODO:
- implement one and not the other
- use tabs
- more configuration like spaces/tabs/column width (Python pprint)
- implement nothing, only change in roblox-cli
- custom metamethod for printing/pretty
- no trailing comma
