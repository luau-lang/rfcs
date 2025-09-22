# string.build() function proposal

## Summary

I propose to add a `string.build` function to provide a convenient API for multi-argument string concatenation, addressing the usability issues of the current `table.concat` approach while maintaining linear time complexity.

## Motivation

String concatenation in Luau using the `..` operator has quadratic time complexity (`O(n²)`) when used repeatedly, creating significant performance issues in string-heavy code.

Performance testing demonstrates the severity of this problem:

| Iterations | Naive `..` | `table.concat` | Performance Gap |
|------------|------------|----------------|-----------------|
| 1,000      | 2.1ms      | 0.2ms          | **10.5x slower** |
| 5,000      | 35.4ms     | 0.8ms          | **44.3x slower** |
| 10,000     | 137.7ms    | 1.5ms          | **91.8x slower** |
| 20,000     | 503.9ms    | 3.3ms          | **152.7x slower** |

For example, a developer may have such logic:

```lua
local result = ""
for i = 1, 10000 do
    result = result .. "item " .. i .. "\n"
end
```
where processing becomes increasingly slow with large `n`, taking 138ms for 10,000 iterations.

The current workaround requires managing arrays manually with `table.concat` as such:
```lua
local parts = {}
for i = 1, 10000 do
    table.insert(parts, "item ")
    table.insert(parts, tostring(i))
    table.insert(parts, "\n")
end
local result = table.concat(parts)
```
which is efficient at 1.5ms, but it's verbose and requires developers to manually separate values into individual table elements.

This pattern is in many real codebases, such as:
- Message/Text systems that build formatted messages with player names, timestamps, and content
- UI text generation combining player stats, currency formatting, and game state
- Debug output combining object properties, coordinates, and state information
- Command parsing and help text generation for admin tools and game commands

Every single one of these scenarios forces developers to choose between performance and maintainability, which is a choice that shouldn't exist in a modern programming language.

The `table.concat` approach itself forces developers to:
- Constantly think about array management instead of the actual problem they're solving
- Mix `table.insert` calls with direct indexing, creating possible bugs
- Hit a performance cliff immediately and learn obscure optimization techniques

The 5x overhead compared to the optimal `table.concat` is acceptable for the ergonomic benefits, and it's still orders of magnitude faster than naive concatenation. Developers who need maximum performance can still use `table.concat` directly.

## Design

The solution is straightforward: provide what every other modern language provides by default.

`string.build(...: string | number | boolean): string`

This function efficiently concatenates all arguments into a single string, with well-defined type conversions:
- `string` values are used directly without conversion
- `number` values are converted using Luau's standard number-to-string formatting (same as `tostring()`)
- `boolean` values are converted to "true" or "false"

The restricted signature prevents runtime errors and makes the function's behavior predictable. For complex types, explicit `tostring()` conversion maintains transparency and type safety.

The function would internally collect arguments into a table and use `table.concat` for the final join operation, providing the same O(1) concatenation performance while offering a more convenient API for multi-argument scenarios.

This type-restricted approach ensures predictable behavior and catches type errors at compile time rather than runtime. For complex types like Vector3, tables, or userdata, developers must explicitly convert them first:
```lua
string.build("Position: ", tostring(part.Position))
string.build("Players online: ", tostring(#players_table))
```
which maintains type safety and clarity, whereas 
```lua
string.build("Position: ", part.Position)
```
would be caught by the type-checker.

### Examples of correct usage:

**Single-call concatenation:**
```lua
local greeting = string.build("Hello, ", name, "! Welcome to ", game_name, ".")
```
which replaces code like:
```lua
local parts = {}
table.insert(parts, "Hello, ")
table.insert(parts, name)
table.insert(parts, "! Welcome to ")
table.insert(parts, game_name)
table.insert(parts, ".")
local greeting = table.concat(parts)
```

**Building structured output:**
```lua
local function render_user_info(user)
    return string.build(
        "Name: ", user.name, "\n",
        "Level: ", user.level, "\n",
        "Coins: ", user.coins, "\n",
        "Premium: ", user.is_premium and "Yes" or "No"
    )
end
```

**Loop-based building:**
```lua
local function format_leaderboard(player_data)
    local parts = {"TOP PLAYERS\n"}
    for i, player in player_data do
        table.insert(parts, string.build(i, ". ", player.name, " - ", player.score, " points\n"))
    end
    return table.concat(parts)
end
```

**Mixed type handling:**
```lua
local function format_purchase_confirmation(player_name: string, item_name: string, cost: number, new_balance: number)
    return string.build("Player ", player_name, " bought ", item_name, 
                       " for ", cost, " coins. New balance: ", new_balance)
end
```

The function handles mixed types with predictable conversions: strings stay strings, numbers become formatted strings using the same rules as `tostring()`, and booleans become `"true"`/`"false"`.

The primary benefit is the ergonomics: reducing the complexity of multi-argument concatenation from:
```lua
table.concat({tostring(a), tostring(b), tostring(c), tostring(d)})
```
to:
```lua
string.build(a, b, c, d)
```

## Drawbacks

This adds one more function to the string library, increasing the API surface area. The performance overhead means it's not always the optimal choice; developers working with performance-critical code should still use `table.concat` directly.

The function is slower than optimal `table.concat` usage due to:
- Function call overhead on every invocation
- Internal argument processing and varargs handling
- Type conversion operations

There's potential for confusion about when to use `string.build` versus `..` or `table.concat`. The rule is: use `string.build` for convenience when concatenating 3+ values in a single operation, but still structure loops to avoid `O(n²)` behavior.

The restricted type signature means developers must be explicit about converting complex types, adding verbosity:
```lua
string.build("Health: ", tostring(humanoid.Health), "/", tostring(humanoid.MaxHealth))
```

Most importantly, the function doesn't automatically solve the `O(n²)` problem; developers must still understand proper concatenation patterns to avoid performance issues.

## Alternatives

- **Maintain status quo.** The current situation actively harms developer productivity and code quality. Unacceptable.

- **Do nothing.** Continue requiring developers to manually manage tables for efficient concatenation. This leaves the ergonomic issues in place but maintains optimal performance for those who understand the patterns.

- **Optimize the `..` operator.** Make `..` automatically efficient for multiple concatenations. This would require significant runtime changes and might break existing performance expectations of the operator.

- **Add string interpolation syntax.**  A much larger language change requiring parser modifications. While potentially valuable, it doesn't address the programmatic string building use cases.
  
- **Add multiple string building functions.** Include functions like `string.join(separator, ...)`, `string.template(format, ...)`, etc. This would fragment the API and create decision paralysis. A single, well-designed function handles the majority of use cases.

- **Improve `table.concat` ergonomics.** Add helper functions or syntax sugar specifically for the table-building pattern. This would maintain optimal performance while addressing usability concerns.
