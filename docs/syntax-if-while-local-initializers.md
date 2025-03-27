# `if local` and `while local` statements

This RFC is an update and continuation to [if statement initializers](https://github.com/luau-lang/rfcs/pull/23), featuring improved semantics, especially in relation to fallthrough.

## Summary

`if local statements`: Allow `local` identifiers to be bound in `if` statements to improve the ergonomics of extremely common control flow idioms, improve code clarity, reduce scope pollution, and improve the developer experience.

`while local statements`: Allow `local` identifiers to be bound in `while` statements to allow them to be used in the loop's condition (and body), improve code clarity, and improve control flow.

## Motivation

Declaring locally-scoped variables at the point of use in `if` statements simplifies code, better conveys the programmer's intent, and leads to more readable and better understandable program logic. By combining `if` condition initializers with nilchecks, users can easily handle success/empty conditions and expressively declare control flow.

In Luau, an extremely common idiom is for fallible functions to return an optional value: an intended result if the operation succeeds or `nil` if it fails. Users are expected to *`nil`check* this result to handle success and failure/empty cases, and this constitutes a major aspect of control flow. An extremely common example of such is Roblox's `Instance:FindFirstChild`, which returns an `Instance` if one was found or nil otherwise:

```luau
local model = workspace:FindFirstChild("MyModel")
if model ~= nil then
    -- model is bound and not nil
end
-- model is still bound here
```

With `if local` statements, this code may be rewritten as:

```luau
if local model = workspace:FindFirstChild("MyModel") then
    -- model is bound and is not nil
end
-- model is not bound here
```

In such cases, `if local` statements better express the programmer's intent and more clearly state the intended control flow.

In many cases, developers use an expression in an if statement's condition and then immediately use it again in its body. Not only does this result in dense, duplicated code, but it also evaluates the expression twice:

```luau
if folders[folder][file_name].last_updated < now - TWO_DAYS then
    last_updated = folders[folder][file_name].last_updated
end
```

In this case, an `if local` statement with an `in` clause can reduce repetition and greatly improve readability:

```luau
if local update_time = folders[folder][file_name].last_updated 
    in update_time < now - TWO_DAYS 
then
    last_updated = update_time
end
```

A simple indexing operation may not be expensive, but if the user wants to call a function instead, they often must call it twice to achieve the intended behavior or bind it outside the `if` statement and pollute the outer scope.

In this (surprisingly common) example, the user calls `table.find` twice to avoid the `table.remove(t, table.find(t, element))` footgun:

```luau
if table.find(array, element) then
    table.remove(array, table.find(array, element))
end
```

In this case, the user chooses to iterate over the array twice instead of assigning index to a local variable, saving a line of code and a `local` binding.

With an `if local` statement, the user can write the code they desire even more succinctly in a more readable way, without having to iterate over the array twice, nor keep `index` bound unnecessarily in the outer scope:

```luau
if local index = table.find(array, element) then
    table.remove(array, index)
end
```

A significant amount of code (especially on Roblox) needs to do multiple (often nested) checks before executing the condition they want to focus on.

With `if local` stacks (`multi-local`s), such nested checks may be rewritten like:

```luau
local function initializeUi()
    if
        local container: ScreenGui = PlayerGui:FindFirstChild("MainContainer")
        local teamsFrame: Frame = container:FindFirstChild("TeamsFrame")
        local roundHeaderFrame: Frame = container:FindFirstChild("RoundHeaders")
        local currentRoundIndicator: TextLabel = roundHeaderFrame:FindFirstChild("CurrentRoundLabel")
        local lastWinnerLabel: TextLabel = roundHeaderFrame:FindFirstChild("LastRoundWinnerLabel")
    then
        -- every one of the bindings above is guaranteed to exist and the user doesn't have to explicitly nilcheck them in here
    else
        task.wait(1)
        return initializeUi()
    end
end
```

The primary motivation for `if local` statements isn't even in small examples like those above, however, it's how it fits into whole codebases. `if local`s can drastically improve code shape, readability, and the general conciseness and expressiveness of the Luau language.

Take this non-trivial example for instance, where we're trying to read `.csv`s into a data structure. Assume `fs.find` returns a table with fields `file` or `dir` which are optional tables.

```luau
-- find and read data csv files
type CsvData = {
    [string]: {
        { string },
    },
}

local function get_input_csvs(): CsvData
    local input_dir = input.get("input data dir: ")
    local cleaned_input_dir = string.gsub(input_dir, "\n", "")
    if #cleaned_input_dir > 0 then
        input_dir = cleaned_input_dir
    else
        print("forgot to enter a path? try again")
        return get_input_csvs()
    end
    
    local data_dir = fs.find(input_dir).dir
    if not data_dir then
        print("invalid input dir, please try again")
        return get_input_csvs()
    end

    local data: CsvData = {}
    for _, entry in data_dir:entries() do
        if entry.type ~= "File" then 
            continue -- skip dirs
        end 
        local filename = entry.name
        if string.find(filename, "%.csv$") then
            local lines: { string } = {}
            for line_number, line in entry:readlines() do
                local line_split = string.split(line)
                -- last column is usually empty
                if string.gsub(line_split[#line_split], "%s", "") == "" then
                    table.remove(line_split, #line_split)
                end
                table.insert(lines, line_split)
            end
            data[filename] = lines
        end
    end
    return data
end
```

With `if local`s, we can rewrite it to:

```luau
type CsvData = {
    [string]: {
        { string }
    }
}

local function get_input_files(): CsvData
    local input_path = input.get("input data dir: ")
    if local stripped_path = string.gsub(input_path, "\n", "") in #stripped_path > 0 then
        input_path = stripped_path
    else
        print("forget to enter a path? try again")
        return get_input_csvs()
    end

    local data: CsvData = {}
    if local input_dir = fs.find(input_path).dir then
        for _, entry in input_dir:entries() do
            if
                local file = entry in entry.type == "File"
                local filename = file.name in string.find(filename, "%.csv$")
                local lines: { string } = {}
            then
                for line_number, line in file:readlines() do
                    local line_split = string.split(line) -- default splits by commas
                    if -- last_column is (almost always) empty
                        local last_column = line_split[#line_split]
                        in string.gsub(last_column, "%s", "") == ""
                    then
                        table.remove(line_split, #split_lines)
                    end
                    table.insert(lines, line_split)
                end
                data[filename] = lines
            end
        end
    else
        print("invalid data dir, please try again")
        return get_input_files()
    end

    return data
end
```

This example shows how `if local` statements can be used to simplify control flow, enhance readability by defining variables in an `if local` stack header (alongside their checks), and clarify the main purpose of the function.

`while local` statements allow you to set a value that can be checked in the conditional expression of the loop and also be used within the loop body.

```luau
while local line = nextline() do -- iteration stops when nextline() returns nil
    print(line)
end
```

Here's a more useful example that combines `if local`s with `while local`s to read `require` aliases from `.luaurc`s:

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
        -- note that luaurcs might not necessarily contain aliases, so .aliases should be nilchecked
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

This proposal introduces `if local` and `while local` statements, or more precisely, allows `local` bindings to be initialized within `if` and `while` statement declarations respectively. This RFC refers these two features as `if local` and `while local` statements to distinguish their mental models from those of regular `if` and `while` statements—and because that's how users refer to them anyway.

### `if local` statements

An `if local` statement is any `if` statement with one or more `local`-`in` clauses.

A `local`-`in` clause may be defined following the `if` or `elseif` keywords in an `if` statement, and consists of one or more `local` bindings and one optional `in` clause expression to affect the execution condition of the branch.

A `local`-`in` clause may be followed by another `local`-`in` clause and must be eventually terminated by the `then` keyword.

Examples:

```luau
if local identifier = expression() then
end
-- or
if local identifier = expression() in condition() then
end
-- or
if local identifier = expression() then
elseif local differentidentifier = expression() then
end
-- or
if
    local x = foo()
    local y = x.bar
    local z = y.baz in z:IsA("BasePart")
then
    -- x and y must be non-nil, and z must be non-nil and a BasePart
end
-- etc.
```

#### Evaluation semantics

If `local` bindings are provided, then one optional `in` clause may be provided per `local`-`in` clause to partially determine the evaluation condition of the `if/elseif` branch.

- If an `in` clause is not provided, then the evaluation condition of the branch is that the leftmost binding must evaluate not-`nil`.
  - This is roughly similar to the current behavior of calling a multiret function in `if` statement condition (except with a `nil` check instead of a truthiness check) in which the conditional branch will evaluate if the first return of the multiret is truthy.

- If an `in` clause is provided, then the clause must be satisfied ***and*** the leftmost binding must evaluate not-`nil`.
- The `in` clause will not be evaluated if the leftmost binding is `nil`.

Although this behavior somewhat differs from the previous RFC, this is because the purpose of an `if local` initializer is to check if values exist, and if they do, to bind them. Since fallthrough is not allowed by this RFC, there isn't a major usecase for allowing for the main `local` binding to be `nil`.

This makes the behavior of `if local`s with `in` clauses *more consistent* with `if local`s without `in` clauses.

By expecting the leftmost binding to always exist, we can better support the primary usecase (only one binding) by allowing users to omit the `character ~= nil` or `character and` `nil` checks in the following example:

```luau
if local character = player.Character 
    character:FindFirstChildOfClass("Humanoid").Health > 20 
    -- since character is the leftmost binding, it's guaranteed to exist 
    -- and a `character and` or `character ~= nil` check isn't needed
then
    -- do something with character
end
```

#### Multiple binding and `multi-local` semantics

Multiple bindings are allowed in the same `local`-`in` clause and must be separated by commas. Like regular `local a, b = foo, bar` assignments, `bar` is not allowed to refer to the new `a` because it isn't initialized yet.

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

To cleanly handle nested conditional pyramids, multiple `local`-`in` clauses may be stacked in same `if local` statement; these can be referred to as `multi-local`s, `if local` stacks, or more colloquially, `if local` pancakes.

`local`-`in` clauses in an `if local` stack must be separated from one another by whitespace or semicolon. Unlike when initializing multiple bindings in the same `local`-`in` clause, `if local` pancakes are allowed to refer to previous bindings in the stack, and this is their unique selling point and a major motivation for this RFC in general.

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

Each `local`-`in` clause in the above `multi-local` evaluates once-at-a-time and can be thought of syntactic sugar for multiple nested `if local`s; this allows subsequent `local` bindings to refer to former ones without nesting and unnecessarily increasing cognitive complexity.

In fact, the above code is practically equivalent to (and in an implementation, can be expanded to) the nested:

```luau
if local model = hit.Parent then
    if local humanoid = model:FindFirstChildOfClass("Humanoid") then
        if local target_player = Players:GetPlayerFromCharacter(model) in target_player.Team ~= player.Team then
            return hitplayer(target_player, humanoid)
        end
    end
end
```

This makes `if local` stacks an extremely useful feature for mitigating ['pyramids of doom'](https://en.wikipedia.org/wiki/Pyramid_of_doom_(programming)) in `nil`check-heavy codebases.

#### `if local` stack evaluation semantics

If any leftmost `local` binding in an `if local` stack evaluates `nil`, then the entire conditional branch won't execute. This allows for multiple checks on possibly-`nil` values in an intuitive manner without having to `nil`check each individually.

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

`if local` fallthrough (bindings in a prior branch's condition are visible in subsequent branches and their conditions) was included in and was a major motivator for the previous `if statement` initializers RFC. It was decided against due to limited utility, complicating the story of the `if local` feature, and since in other languages, it often leads to unexpected behavior, possible footguns, and is mostly only useful for error catching. Additionally, as proposed in the previous RFC, `if local` fallthrough could easily result in hard-to-understand control flow which could increase the cognitive complexity of Luau code for little benefit.

As an alternative, this RFC's `if local` pancakes provide a better way to handle stacked and dependent conditions without greatly increasing Luau's cognitive complexity.

#### Edge cases

Initializations without assignments (`if local x then end`) are not permitted and cause a syntax error. Initializations of the leftmost binding to `nil`, including but not limited to the following:

- `if local x = nil then end`
- `if local x, y = nil, 3 in x == nil or y then end`

will always evaluate to `false` and will never execute a conditional branch; this means that the existing 'if condition always false' lint should be expanded to include `if local`s that never evaluate. This should reduce the likelihood of users mistakenly writing these never-evaluating `if local`s, and iron out any confusion on requiring the leftmost binding to be not-`nil`.

Type annotations may be provided alongside a `local` binding as expected:

```lua
type Cat = {
    name: string,
    kittens: { Cat },
}

if local cat: Cat in (cats :: { Cat? })[1] then
end
```

As an edge case, locals may be reassigned within the `in` condition. In this case, the conditional branch executes because x is initially not `nil`, allowing the `in` condition to evaluate, reassign `x` to `nil`, and return `true`:

```luau
if local x = 3 in (function() x = nil; return true end)() then
    print(x) -- nil
end
-- or
if local x: number = foo(); local y: number = (function() local y = x + 1; x = nil; return y end)() then
    print(typeof(x), typeof(y)) --> nil, number
end
```

### `while local loops`

`while local` statements are defined as `while` loops with one or more `local` bindings. Similarly to `if local`s, each `local-in` clause must include one or more `local` bindings declared after the `while` keyword, may contain one `in` conditional clause, may be followed by other `local`-`in` clauses and must end finally with the `do` keyword.

```luau
while local identifier = expr() do
end
-- or
while local identifier = expr() in cond() do
end
-- or
while
    local x = foo()
    local b = if typeof(x) == "number" then bar() else baz() in isvalid(b)
do
end
-- etc.
```

`while local` identifiers are initialized and assigned once before for every iteration of the while loop, and are visible in their `in` conditions. Similarly to `if local`s, stacked `local`s in `while local`s are allowed, and iteration stops if any `in` clause evaluates falsey.

Note that unlike with `if local` stacks, implementing `while local` stacks might be more complicated than just by nesting multiple control structures. We want to propose this within the RFC and let a Luau language engineer with better knowledge of the Luau (non-pancake) stack judge their viability in comparison to `if local` stacks.

```luau
while local x = nextline() in x ~= "" then
end
-- or
local contents: { string } = {}
while
    local x = nextline() in x ~= ""
    local content = x:match("content: ([%w%s%p]+)\n")
do
    table.insert(contents, content)
end
```

## Drawbacks / Arguments against

- You can already define `local`s outside a control structure and scope them in.

Yes, but we want `if local` and `while local`s! They will make the language a lot more expressive and cohesive!

- `if local`s increase the cognitive complexity of the language and aren't enough of a value-add to implement.

While `if local`s may increase the learning burden of the language for beginners (and for existing users upon this syntax's release), they can increase the readability of Luau code, decrease the cognitive complexity of new code in Luau over time, and allow for simple and really common idioms to be expressed in a more concise way friendly for users. Additionally, `if local`s (and to a lesser extent, `while local`)s are relatively popular additions to Luau and should be received in a very friendly way by existing users.

- Adding `if local` statements without `if local` *expressions* will be confusing to users

This is a viable concern, however we feel the benefits of adding `if local` statements without an expression equivalent outweigh the drawbacks of not having `if local` statements at all. `if` expressions already have some differences from `if` statements (they can't retun multirets for example and *must* end with an `else`), so adding no `local`s to that mental model shouldn't be too drastic of a drawback. Furthermore, `if local` expressions may be introduced in the future if generalized bindings-in-expression syntax is agreed upon and a subsequent compiler refactor is deemed necessary to allow for it.

## Alternatives

- The leftmost binding should not be required to be non-`nil` when an `in` clause is provided.

With this alternative, users would have to specify `identifier and` in every case they want to use `identifier` in an `in` clause.

Since a primary motivator for `if local`s is `nil`checking, and the default behavior of `if local x = foo()` is that `x` should be non-nil, we argue that it makes more sense (and is more intuitive) for `if local`s to always assume `x` is non-nil, even when an `in` clause is provided.

- the default should be for ALL bindings to be non-`nil`, or require users write an `in` condition for `if local`s with multiple bindings within the same `local`-`in` clause.

If all bindings were non-nil, we could hit unexpected behavior where users expect their conditions to evaluate but they never do. Additionally, it's common for `pcall` to not return a value if it succeeds.

```luau
if local success, err = pcall(function()
    some_roblox_datastore:SetAsync(userid, data)
end) then
    if success then
        -- handle successs
    else
        -- handle fail
    end
end
```

this would never evaluate the conditional branch because `err` would always be nil.

As a result of this, it's better to stick to where the first `local` cannot be nil, but others can and can be explicitly checked if desired.

Additionally, the fact that `multi-local`s exist means that we should guide users towards `multi-local`s instead of initializing multiple bindings in one `local`-`in` clause, which is preferable than just requiring the `in` clause to be present when multiple bindings are declared.

- `if` statement initializers don't necessarily need the `local` keyword to be parseable:

```luau
if x = foo() in x and x:match("hi") then
end
```

The main reason for adopting `if local` syntax over `if identifier =` syntax is because `if local`s are more visually distinct with syntax highlighting, and therefore easier for humans to parse than `if identifier =` syntax, and because `if local`s can be considered a different user-facing feature than `if` statements, so should require a slightly different mental model.

A counterargument to this is that since the existing numerical loop syntax (`for i = 1, 10 do`) has an assignment without the the `local` keyword, so should `if` statement initializers to keep the language internally consistent.

We feel that the usecases sufficiently differ (`nil`checking vs numerical iteration), and that `if local` syntax is sufficiently different from numerical for loops that users will easily be able to adopt their mental models to distinguish between them and `if/while local` statements.

It is notable that Rust requires `let` in both `if let` and `while let` expressions but doesn't require `let` in `for` loops, therefore having asymmetric `for` loops and `if/while`s when the latter introduces bindings isn't unheard of outside Luau.

- Another proposed alternative was for dropping the `in` keyword for allowing the `local x = foo()` binding to be used as an expression like the walrus operator (`:=`) in Python:

```luau
if local a = foo() and a:match("hi") then
end
```

The issue here is ambiguity with operator precedence. If we follow Python's implementation of the walrus, then `a` will evaluate to `foo() and a:match("hi")` (a boolean) (or break as `a` (probably `nil` unless it's a shadow) doesn't have a method `match`) instead of what the user probably intended. For this to work as intended, the `=` operator would have to bind extremely tightly to its RHS, but only in `if local`s. This would require a more complicated mental model for `if` statement initializers and would not be very user-friendly.

Additionally, Luau does not and (for now) will not support generalized bindings-in-expression syntax. Allowing `local = foo()` to be used as an expression, but only within `if` statement conditions, would certainly complicate the mental model around expressions in Luau.

We have decided that separating the assignment expression and the evaluation expression with an `in` clause is the best way to express the logic users intend with `if local`s.

- Have `if identifier = expr()` alongside `if` and `while` locals.

In this case, `if` statement initializers would allow for fallthrough whereas `if local`s wouldn't.

This alternative would allow:

```luau
if content_type = headers["Content-Type"] or headers["content-type"] 
    in content_type == "text/plain" or not content_type 
then
    handle_plaintext(body)
elseif content_type == "application/json" then
    handle_json(body)
elseif content_type == "application/octet-stream" then
    handle_bytes(body)
elseif content_type == "image/png" or content_type == "image/jpeg" then
    handle_image(body)
elseif local override_body = override_bodies[content_type] then
    handle_custom(override_body, content_type)
else
    error(`encountered unexpected content-type {content_type}`)
end
-- content_type not visible here
```

This would provide parity with `for i = 1, 10 do` syntax, and would appease both the fallthrough camp and the non-fallthrough camp. However, we don't see a huge reason to add a separate syntax just for fallthrough, when the primary purpose of `if local`s is for the common idiom of `nil`checking. If a user wants fallthrough, they can just define the variable above the control structure as they do currently.

Additionally, a major usecase for `if local` fallthrough was for handling early returns by unnesting heavily-nested conditional structures, a usecase that's satisfied, in our opinion, more elegantly by `if local` stacks.

## Future work

`if local` *expressions* are not formally included in this RFC due to implementation difficulty (specifically, a sizeable compiler rewrite), however are included as a future proposal and would follow extremely similar syntax to `if local` statements:

```luau
local humanoid: Humanoid? = if local character = player.Character
    then character:FindFirstChildOfClass("Humanoid")
    else nil
```

For `if local` *expressions* to be possible, Luau would need bindings-in-expressions support, which would require a compiler rewrite.
