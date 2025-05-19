# `if local` and `while local` statements

This RFC is an update and continuation to [if statement initializers](https://github.com/luau-lang/rfcs/pull/23), featuring improved semantics, especially in relation to fallthrough.

## Summary

`if local statements`: Allow `local` identifiers to be bound in `if` statements to improve the ergonomics of extremely common control flow idioms, improve code clarity, reduce scope pollution, and improve the developer experience.

`while local statements`: Allow `local` identifiers to be bound in `while` statements, re-evaluated upon iteration, and visible in the loop's evaluation condition and `do` body to provide for better control flow.

## Motivation

In Luau, an extremely common idiom is for fallible functions to return an optional value. Users are expected to *`nil`check* (or truth-check) this result to handle success and failure/empty cases, and this constitutes a major aspect of control flow.

Declaring locally-scoped variables at the point of use in `if` statements simplifies code, better conveys programmer intent, and leads to more readable and better understandable program logic. By combining `if` condition initializers with truthiness checks, users can easily handle success/empty conditions and expressively declare control flow.

An extremely common example of such is Roblox's `Instance:FindFirstChild`, which returns an `Instance` if one was found or `nil` otherwise:

```luau
local model = workspace:FindFirstChild("MyModel")
if model then
    -- model is bound and not nil/false
end
-- model is still bound here
```

`if local` statements can simplify this code to:

```luau
if local model = workspace:FindFirstChild("MyModel") then
    -- model is bound here and not nil/false
end
-- model is not bound here
```

In many cases, developers use an expression in an if statement's condition and then immediately use it again in its body. Not only does this result in dense, duplicated code, but it also evaluates the expression twice:

```luau
if folders[folder][file_name].last_updated < now - TWO_DAYS then
    last_updated = folders[folder][file_name].last_updated
end
```

In this case, an `if local` statement with an `in` clause can reduce repetition and greatly improve readability:

```luau
if local update_time = folders[folder][file_name].last_updated 
    in update_time and update_time < now - TWO_DAYS 
then
    last_updated = update_time
end
```

A simple indexing operation may not be expensive, but if the user wants to call a function instead, they often must call it twice to achieve the intended behavior or bind it outside the `if` statement.

In this (surprisingly common) example, the user calls `table.find` twice to avoid the `table.remove(t, table.find(t, element))` footgun:

```luau
if table.find(array, element) then
    table.remove(array, table.find(array, element))
end
```

With an `if local` statement, the user can rewrite the code without iterating over the array twice, nor keeping `index` bound unnecessarily in the outer scope:

```luau
if local index = table.find(array, element) then
    table.remove(array, index)
end
```

A significant amount of code (especially on Roblox) needs to do multiple (often nested) checks before executing the condition they want to focus on.

With `if local` stacks, such nested checks may be rewritten like:

```luau
local function initializeUi()
    if
        local container: ScreenGui = PlayerGui:FindFirstChild("MainContainer")
        local teamsFrame: Frame = container:FindFirstChild("TeamsFrame")
        local roundHeaderFrame: Frame = container:FindFirstChild("RoundHeaders")
        local currentRoundIndicator: TextLabel = roundHeaderFrame:FindFirstChild("CurrentRoundLabel")
        local lastWinnerLabel: TextLabel = roundHeaderFrame:FindFirstChild("LastRoundWinnerLabel")
    then
        -- every one of the bindings above is guaranteed to exist and the user doesn't have to explicitly check them in here
    else
        task.wait(1)
        return initializeUi()
    end
end
```

Basically, `if local`s can drastically improve code shape, readability, and the general conciseness and expressiveness of the Luau language.

`while local` statements instantiate bindings which are visible in the conditional evaluation expression of the loop as well as the loop body. These can be easily compared to `while let` loops in Rust, which serve a similar purpose.

Here's a useful example that combines `if local`s with `while local`s to read `require` aliases from `.luaurc`s:

```luau
local luaurcs_found: { string } = {}
local aliases: { [string]: string } = {}
local current_path = get_requiring_file_path()
while local parent_path = path.parent(current_path) do
    -- stops iterating when path.parent returns nil (we've reached the filesystem root)
    if 
        local luaurc_path = path.join(parent_path, ".luaurc")
        local luaurc = fs.find(luaurc_path).file
    then
        local data = json.decode(luaurc:read())
        -- note that luaurcs might not necessarily contain aliases, so .aliases should be existence checked
        if local found_aliases = data.aliases then
            for alias, to_path in found_aliases do
                if not aliases[alias] then
                    aliases[alias] = to_path
                end
            end
        end
        table.insert(luaurcs_found, luaurc_path)
    end
    current_path = parent_path
end
```

## Design

This proposal introduces `if local` and `while local` statements, or more precisely, allows `local` bindings to be initialized within the conditional expression portions of `if` and `while` statement declarations respectively.

Note that this RFC refers these two features as `if local` and `while local` statements to distinguish their mental models from those of regular `if` and `while` statements—and because it's how users refer to them anyway.

The grammar for `if` and `while` statements shall be changed to the following:

```ebnf
stat ::= varlist '=' explist |
    ...
    'while' ifwhilecond 'do' block 'end' |
    ...
    'if' ifwhilecond 'then' block {'elseif' ifwhilecond 'then' block} ['else' block] 'end' |
...
ifwhilecond ::= exp | {'local' bindinglist '=' explist ['in' exp][';']}
```

### `if local` statements

An `if local` statement is any `if` statement with one or more `local`-`in` clauses.

A `local`-`in` clause may be defined following the `if` or `elseif` keywords in an `if` statement, and consists of one or more `local` bindings and one optional `in` clause expression to set the execution condition of the branch.

A `local`-`in` clause may be followed by another `local`-`in` clause and must be eventually terminated by the `then` keyword.

Examples:

```luau
if local identifier = expression() then
end
-- or
if local identifier = expression() in condition() then
elseif local different_identifier = expr() then
end
-- or
if
    local x = foo()
    local y = x.bar
    local z = y.baz in z and z:IsA("BasePart")
then
    -- x and y must be truthy, and z must be truthy and a BasePart
end
-- etc.
```

#### Evaluation semantics

If `local` bindings are provided, then one optional `in` clause may be provided per `local`-`in` clause to partially determine the evaluation condition of the `if/elseif` branch.

- If an `in` clause is not provided, then the evaluation condition of the branch is that the leftmost binding must evaluate truthy; this is the same as the current behavior of calling a multiret function within an `if` statement's condition.
- If an `in` clause is provided, then the default truthiness check is overriden by the `in` clause expression.

<!-- Although this behavior somewhat differs from the previous RFC, this is because the purpose of an `if local` initializer is to check if values exist, and if they do, to bind them. Since fallthrough is not allowed by this RFC, there isn't a major usecase for allowing for the main `local` binding to be `nil`.

This makes the behavior of `if local`s with `in` clauses *more consistent* with `if local`s without `in` clauses. This also allows us to prioritize the most common usecase (a single `local` binding), allowing users to omit the `character ~= nil` or `character and` `nil` checks in the following example:

```luau
if local character = player.Character 
    in character:FindFirstChildOfClass("Humanoid").Health > 20 
then
    -- do something with character
end
``` 
-->

#### Multiple bindings and `if local` stack semantics

Multiple bindings are allowed in the same `local`-`in` clause and must be separated by commas. Like regular `local a, b = foo, bar` assignments, `bar` is not allowed to refer to the new `a`.

```luau
if local success, result = pcall(foo) then
    -- note that success can be false here and still bound; false is falsey but not nil!
    if success then
        dothing(result)
    else
        print(result)
    end
end
-- or
if local nevernil, maybenil = foo() in maybenil ~= nil then
    -- nevernil and maybenil are both guaranteed to be non-nil
end
```

To cleanly handle nested conditional pyramids, multiple `local`-`in` clauses may be stacked in same `if local` statement; these can be referred to as `if local` stacks, or more colloquially, `if local` pancakes.

`local`-`in` clauses in an `if local` stack must be separated by whitespace or semicolon. Unlike when initializing multiple bindings in the same `local`-`in` clause, `if local` pancakes are allowed to refer to previous bindings in the stack, and this is their unique selling point and a major motivation for this RFC in general.

To demonstrate the utility of this, here's a simple Roblox example:

```luau
if
    local model = hit.Parent
    local humanoid = model:FindFirstChildOfClass("Humanoid")
    local target_player = Players:GetPlayerFromCharacter(model) in target_player.Team ~= player.Team
then
    return hitplayer(target_player, humanoid)
end
```

`if local` stacks can be thought of syntactic sugar for unnesting `if local`s; this allows subsequent `local` bindings to refer to former ones without unnecessarily increasing cognitive complexity.

The above code is practically equivalent to (and in an implementation, can be expanded to) the nested:

```luau
if local model = hit.Parent then
    if local humanoid = model:FindFirstChildOfClass("Humanoid") then
        if local target_player = Players:GetPlayerFromCharacter(model) in target_player.Team ~= player.Team then
            return hitplayer(target_player, humanoid)
        end
    end
end
```

This is an extremely useful feature for mitigating ['pyramids of doom'](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)) in `nil`check-heavy codebases.

#### `if local` stack evaluation semantics

If any leftmost `local` binding in an `if local` stack evaluates falsey, then the entire conditional branch won't execute. This allows for multiple checks on possibly-falsey values in an intuitive manner without having to check each individually.

#### Binding semantics

Variables bound in an `if local` initializer remain in scope within their `in` clause condition and `then` body, and subsequently go out of scope before the next conditional branch or `end`. In other words, `if local` bindings have *no fallthrough*.

For example,

```luau
if local cats = getcats() :: { Cat } in #cats > 0 then
    -- cats is bound here
elseif local dogs = getdogs() :: { Dog } in #dogs > 0 then
    -- dogs is bound here, but cats isn't
else
    -- neither cats nor dogs is bound here
end
-- neither cats nor dogs is bound here
```

#### Fallthrough

`if local` fallthrough (bindings in a prior branch's condition are visible in subsequent branches conditions and their `then` bodies) was included in and was a major motivator for the previous `if statement` initializers RFC. It was decided against due to limited utility, complicating the story of the `if local` feature, and since in other languages, it often leads to unexpected behavior, possible footguns, and is mostly only useful for error catching.

As an alternative, this RFC's `if local` stacks provide a better way to handle stacked and dependent conditions without greatly increasing Luau's cognitive complexity.

#### Edge cases

Initializations without assignments (`if local x then end` or `if local x: FooType then end`) are not permitted and cause syntax errors.

Locals may be reassigned within the `in` condition. In this case, the conditional branch executes because x is initially not `nil`, allowing the `in` condition to evaluate, reassign `x` to `nil`, and return `true`:

```luau
if local x = 3 in (function() x = nil; return true end)() then
    print(x) -- nil
end
-- or
if local x: number = foo(); local y: number = (function() local y = x + 1; x = nil; return y end)() then
    print(typeof(x), typeof(y)) --> nil, number
end
```

### `while local` loops

`while local` statements are defined as `while` loops with one or more `local` bindings. Similarly to `if local`s, each `local-in` clause must include one or more `local` bindings declared after the `while` keyword, may contain one `in` conditional clause, may be followed by other `local`-`in` clauses and must be eventually followed with the `do` keyword.

```luau
while local line = nextline() do
end
-- or
while local text = gettext() in text and text ~= "" do
end
-- or
while
    local x = foo()
    local b = if typeof(x) == "number" then bar() else baz() in isvalid(b)
do
end
-- etc.
```

`while local` identifiers are reevaluated before every iteration of the while loop and are visible in their `in` conditions and loop body. Similarly to `if local`s, stacked `local`s in `while local`s are allowed, and iteration stops if any `in` clause evaluates falsey.

> Note that unlike with `if local` stacks, implementing `while local` stacks might be more complicated than by just nesting multiple control structures. We want to propose this within the RFC and let a Luau language engineer with better knowledge of the Luau (non-pancake) stack and compiler judge their viability in comparison to `if local` stacks.

## Prior art

### Rust `if let`

Rust's `if let` expressions are the primary influence for `if local`s, and allow identifiers to be declared within an `if` expression's condition. Additionally, `if let` chains allow multiple variables to be declared in the same manner. `if let` branches have no fallthrough.

```rust
if let Some(a) = vec_deque.pop_front() && a.ends_with(".luau") {
    println!(a);
}
```

### Python walrus operator

Python allows identifier assignment in expressions with its `:=` (walrus) operator. Due to operator precedence ambiguity, this often means users need to enclose every use of it with parentheses for the intended result. We want to avoid this with `if local`s.

### use of `=` within expressions

Many languages like Ruby, etc. allow the use of identifier assignments as an expression evaluating to the RHS.

## Drawbacks / Arguments against

- You can already define `local`s outside a control structure and scope them in.

Yes, but we want `if local` and `while local`s! They will make the language a lot more expressive and cohesive!

- `if local`s increase the cognitive complexity of the language and aren't enough of a value-add to implement.

While `if local`s may increase the learning burden of the language for beginners (and for existing users upon this syntax's release), they can increase the readability of Luau code, decrease the cognitive complexity of new code in Luau over time, and allow for simple and really common idioms to be expressed in a more concise way friendly for users.

- Adding `if local` statements without `if local` *expressions* will be confusing to users

This is a viable concern, however we feel the benefits of adding `if local` statements without an expression equivalent outweigh the drawbacks of not having `if local` statements at all. `if` expressions already have some differences from `if` statements (they can't retun multirets for example and *must* end with an `else`), so adding no `local`s to that mental model shouldn't be too drastic of a drawback. Furthermore, `if local` expressions may be introduced in the future if generalized bindings-in-expression syntax is agreed upon and a subsequent compiler refactor is deemed necessary to allow for it.

## Alternative: `local` chains

Instead of associated `in` expressions to specific `local` bindings, we could allow mixture of `local` bindings and expressions in a similar way to Rust's `let` chains. This would require a different operator/separator than `and` to preserve backwards compatibility, but would greatly simplify the mental model around `if local`s.

For example, using `&&` as the separator (it's what's used in Rust and isn't a Luau operator yet):

```luau
local foo: () -> string?
local boo: () -> string?

if local a = foo() && local b == boo() && a == b then
end
```

Since `if local`s already test truthiness, by the time the `a == b` clause evaluates both `a` and `b` are guaranteed to be truthy.

For a more useful example (Roblox-related):

```luau
if local player = Players:GetPlayerFromCharacter(model)
    && player.Team ~= Players.LocalPlayer.Team
    && local character = player.Character
    && local humanoid = character:FindFirstChildOfClass("Humanoid")
    && humanoid.Health < 25
then
    return "Red"
end
```

## Previously considered alternatives

Many alternate proposals and alternative semantics were considered for this RFC. In no particular order:

### The leftmost binding should be required to be truthy *even* when an `in` clause is provided

With this alternative, users would not have to specify `identifier and` in every case they want to use `identifier` in an `in` clause.

For example:

```luau
if local humanoid = character:FindFirstChildOfClass("Humanoid")
    in humanoid.Health <= 20 -- `humanoid and` check not needed
then
end
```

`local` chains seem like a better solution to this.

### the default should be for ALL bindings to be truthy or require users write an `in` condition for `if local`s with multiple bindings within the same `local`-`in` clause

This would decrease the ergonomics of using `if local` with `pcall`.

Additionally we should guide users towards `if local` stacks instead of initializing multiple bindings in one `local`-`in` clause, which is more preferable than just requiring an `in` clause be present when multiple bindings are declared.

### `if` statement initializers don't necessarily need the `local` keyword

```luau
if x = foo() in x and x:match("hi") then
end
```

The main reason for adopting `if local` syntax over `if identifier =` syntax is because `if local`s are more visually distinct and therefore easier for humans to parse than `if identifier =` syntax, and because `if local`s require a slightly different mental model than `if` statements. Not requiring `local` can also cause a footgun of confusion between `==` and `=` as both would be valid within an `if` condition.

A counterargument to this is that since the existing numerical loop syntax (`for i = 1, 10 do`) has an assignment without the the `local` keyword, so should `if` statement initializers to keep the language internally consistent.

We feel that the usecases sufficiently differ (`nil`checking vs numerical iteration), and that `if local` syntax is sufficiently different from numerical for loops that users will easily be able to distinguish between them and `if/while local` statements.

Note that Rust requires `let` in both `if let` and `while let` expressions but doesn't require `let` in `for` loops, therefore having asymmetric `for` loops and `if/while`s syntax when the latter introduces bindings isn't unheard of outside Luau.

### Drop the `in` keyword for allowing the `local x = foo()` binding to be used as an expression like the walrus operator (`:=`) in Python

```luau
if local a = foo() and a:match("hi") then
end
```

The issue here is ambiguity with operator precedence. If we follow Python's implementation of the walrus, then `a` will evaluate to `foo() and a:match("hi")` (a boolean) (or break as `a` (probably `nil` unless it's a shadow) doesn't have a method `match`) instead of what the user probably intended. For this to work as intended, the `=` operator would have to bind extremely tightly to its RHS, but only in `if local`s. This would require a more complicated mental model for `if` statement initializers and would not be very user-friendly.

Additionally, Luau does not and (for now) will not support generalized bindings-in-expression syntax. Allowing `local = foo()` to be used as an expression, but only within `if` statement conditions, would certainly complicate the mental model around expressions in Luau. Explicit `in` clauses are clearly superior to those options.

### Have `if identifier = expr()` with fallthrough alongside `if` and `while` locals without fallthrough

In this case, `if` statement initializers would allow for fallthrough whereas `if local`s wouldn't.

This alternative would allow:

```luau
if content_type = headers["Content-Type"] or headers["content-type"] 
    in content_type == "text/plain" or not content_type 
then
    handle_plaintext(body)
elseif content_type == "application/json" then
    handle_json(body)
elseif local override_body = override_bodies[content_type] then
    handle_custom(override_body, content_type)
else
    error(`encountered unexpected content-type {content_type}`)
end
-- content_type not visible here
```

This would provide parity with `for i = 1, 10 do` syntax, and would appease both the fallthrough camp and the non-fallthrough camp. However, this seems unnecessary, and we don't see a huge reason to add a separate syntax just for fallthrough. If a user wants fallthrough, they can just define the variable above the control structure as they do currently.

Furthermore, most usecases for fallthrough are satisfied more elegantly, in our opinion, by `if local` stacks.

### Alternative keywords for `in`

Last RFC proposed and rejected `where` and a semicolon `;` as alternatives to `in`. In the previous RFC, `where` was rejected for behaving in the opposite fashion of `where` in Rust and OCaml in which bindings where declared *after* the `where` clause instead of before. This is still in contention, as most Luau users haven't used Rust or OCaml before, and to many users, `where` seems like a more reasonable keyword there to its use in the English language (as well as SQL). The semicolon was rejected because it's easy to miss and isn't coherent with the keywords-not-symbols philosophy of Luau's runtime syntax.

## Future work

`if local` *expressions* are not formally included in this RFC due to implementation complexity (specifically, a sizeable compiler rewrite), however are included as a future proposal and would follow extremely similar syntax and semantics to `if local` statements:

```luau
local humanoid: Humanoid? = if local character = player.Character
    then character:FindFirstChildOfClass("Humanoid")
    else nil
```
