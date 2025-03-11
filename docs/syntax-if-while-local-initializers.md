# `if local` and `while local` statements

This RFC is an update and continuation to [if statement initializers](https://github.com/luau-lang/rfcs/pull/23), featuring improved semantics as discussed and agreed-upon in that RFC's thread and in ROSS.

## Summary

`if local statements`: Allow `local` identifiers to be bound in `if` statements to improve the ergonomics of extremely common control flow idioms, improve code clarity, reduce scope pollution, and improve the developer experience.

`while local statements`: Allow `local` identifiers to be bound in `while` statements to improve code clarity, improve sentinel value handling semantics, reduce scope pollution, and provide parity with `if local` statements.

## Motivation

Declaring locally-scoped variables at the point of use in `if` statements simplifies code, better conveys the programmer's intent, and leads to more readable and better understandable program logic. By combining `if` condition initializers with nilchecks, users can easily handle success/empty conditions and expressively declare control flow.

In Luau, an extremely common idiom is for fallible functions to return an optional value: an intended result if the operation succeeds or `nil` if it fails. Users are expected to *nilcheck* this result to handle success and failure/empty cases, and this constitutes a major aspect of control flow. An extremely common example of such is Roblox's `Instance:FindFirstChild`, which returns an `Instance` if one was found or nil otherwise:

```luau
local model = workspace:FindFirstChild("MyModel")
if model then
    -- model is bound and not nil
end
-- model is still bound here
```

With `if local` statements, this code may be rewritten:

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
in update_time < now - TWO_DAYS then
    last_updated = update_time
end
```

A simple indexing operation may not be expensive, but if the user wants to call a function instead, they often must call it twice to achieve the intended behavior. In this common example, the user calls `table.find` twice to avoid the `table.remove(t, table.find(t, element))` footgun:

```luau
if table.find(array, element) then
    table.remove(array, table.find(array, element))
end
```

In this case, the user chooses to iterate over the array twice instead of assigning index to a local variable, saving a line of code and a `local` binding. With an `if local` statement, the user can write the code they desire even more succinctly, without having to iterate over the array twice, nor keep `index` bound unnecessarily in the outer scope:

```luau
if local index = table.find(array, element) then
    table.remove(array, index)
end
```

The primary motivation for `if local` statements isn't even in small examples like those above, however, it's how it fits into whole codebases. `if local`s drastically improve code shape, readability, and the general conciseness and expressiveness of the Luau language.

Take this non-trivial example for instance. Assume `fs.find` returns a table with methods like `:exists()` and fields `file` or `dir` which contain optional tables. Assume `path.parent` and `path.child` both return `string?`.

```luau
-- find and read data csv files
type CsvData = {
    [string]: {
        { string },
    },
}

local function get_input_csvs(): CsvData
    local data_dir = input.get("input data dir: ")
    if #string.gsub(data_dir, "\n", "") == 0 then
        return get_input_csvs()
    end
    
    local input_dir = fs.find(cleaned_data_dir).dir
    if not input_dir then
        print("invalid input dir, please try again")
        return get_input_csvs()
    end

    local data: CsvData = {}
    for _, entry in input_dir:entries() do
        if entry.file and string.match(entry.name, "[%.]csv$") then
            local lines: { string } = {}
            for line_number, line in entry.file:readlines() do
                if line_number == 1 then continue end -- skip headers
                local line_split = string.split(line)
                -- last is usually empty
                if string.gsub(line_split[#line_split], " ", "") == 0 then
                    table.remove(line_split, #line_split)
                end
                table.insert(lines, line_split)
            end
            data[entry.name] = lines
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
    local data_dir = input.get("input data dir: ")
    if #string.gsub(data_dir, "\n", "") == 0 then
        return get_input_files()
    end

    if local input_dir = fs.find(data_dir).dir then
        local data: CsvData = {}
        for _, file in input_dir:entries() do
            if local file = entry.file in string.match(file.name, "[%.]csv$") then
                local lines: { string } = {}
                for line_number, line in file:readlines() do
                    if line_number == 1 then continue end -- skip headers
                    local line_split = string.split(line)
                    -- last split is usually empty
                    if string.gsub(line_split[#line_split], " ", "") == 0 then
                        table.remove(line_split, #line_split)
                    end
                    table.insert(lines, line_split)
                end
                data[file.name] = lines
            end
        end
        return data
    end

    print("invalid input dir, please try again")
    return get_input_csvs()
end
```

This example shows how `if local` statements can be used to simplify control flow and clarify the main purpose of the function. It also shows the correct way to use `if local`s, to simplify control flow without simply replacing every `if` statement with an `if local`.

<!-- TODO: add evidence -->

`while local` statements allow you to set a value that persists throughout the execution of the loop as well as an `in` condition that re-evaluates every iteration of the loop.

In this case, `current_path` persists throughout all iterations, and without an `in` clause, the `while local` defaults to `while true do` behavior:

```luau
local init_luau_path = ""
while local current_path = provided_path do
    if local init_luau = fs.find(path.join(current_path, "init.luau")).file then
        init_luau_path = init_luau
        break
    elseif local parent = path.parent(current_path) then
        current_path = parent
    else
        error("ran out of parents")
    end
end
```

In another case, this `while local` loop allows for easy request retries:

```luau
local result
local retries = 0

while local response = http.get("https://my.unreliable.dev/api/") 
in 
    response.status_code ~= 200
    and retries < 3 
do
    if response.status_code == 429 then
        task.wait(response.headers["Retry-After"] or 3)
    elseif response.status_code == 404 then
        break
    end
    
    local new_response = http.get("https://my.unreliable.dev/api/")
    if new_response.status_code == 200 then
        result = new_response.body
    else
        retries += 1
        response = new_response
    end
end
```

## Design

This proposal introduces `if local` and `while local` statements, or more precisely, allows `local` bindings to be initialized within `if` and `while` statement declarations.

### `if local` statements

An `if local` statement is any `if` statement with one or more `local` bindings. `local` bindings may be declared after the `if` or `elseif` keywords of an `if` statement, may be followed by one `in` clause expression, and must be followed by the `then` keyword:

```luau
if local identifier = expression() then
end
-- or
if local identifier = expression() in condition() then
end
```

If `local` bindings are provided, then one optional `in` clause may be provided per branch to partially determine the evaluation condition of the `if/elseif` branch.

- If an `in` clause is not provided, then the evaluation condition of the branch is that the leftmost binding must evaluate not-`nil`. This is roughly similar to the current behavior of putting a multiret function call in an `if` statement condition; the conditional branch will evaluate if the first return of the multiret is truthy.

<!-- - If an `in` clause is provided, then that clause must be satisfied ***in addition*** to all bindings being not-`nil`. Note that the `in` clause will never be evaluated if *any* `local` binding evaluates to `nil`. -->

- If an `in` clause is provided, then the clause must be satisfied and the leftmost binding must evaluate not-`nil`. The `in` clause will not be evaluated if the leftmost binding is `nil`.

Although this behavior somewhat differs from the previous RFC, this is because the purpose of an `if local` initializer is to check if values exist, and if they do, to bind them. By expecting the leftmost binding to always exist, we can better support the primary usecase (only one binding) and allow users to omit the `character and` check in the following `in` clause:

```luau
if local character = player.Character 
in character:FindFirstChildOfClass("Humanoid").Health > 20 then
-- since character is the leftmost binding, it's guaranteed to exist
end
```

Initializations without assignments (`if local x then end`) are not permitted and cause a syntax error. Initializations of the leftmost binding to `nil`, including but not limited to the following:

- `if local x = nil then end`
- `if local x, y = nil, 3 in x == nil or y then end`

will always evaluate to `false` and will never execute a conditional branch.

Multiple bindings are allowed and must be separated by commas:

```luau
if local success, result = pcall(foo) then
-- note that success can be false here and still bound; false is falsey but not nil!
    if success then
        dothing(result)
    else
        print(result)
    end
end
```

> Consider: can/should we allow locals split by semicolons/whitespace like in:
>
> ```luau
> if
>     local x, y = foo()
>     local entry = fs.find("idk.txt")
>     local file = entry.file
> in
>     entry:exists()
>     and file ~= nil
>     and file:read():match(`{x}{y}`) 
> then
>     print("yes")
> end
> ```
>
> This could make `if local`s with multiple bindings a lot more readable, the issue is just if it's even possible. `if local`s are possible by special casing `if` statements without needing generalized bindings-in-expressions, but what about these? I assume it'd work if we disallowed locals from referring to each other.

Variables bound in an `if local` initializer remain in scope within their `in` clause condition and `then` body, and subsequently go out of scope before the next conditional branch or `end`. In other words, `if local` bindings have *no fallthrough*.

For example,

```luau
if local cats = getCats() :: { Cat } in #cats > 0 then
    -- cats is bound here
elseif local dogs = getDogs() :: { Dog } in #dogs > 0 then
    -- dogs is bound here, but cats isn't
else
    -- neither cats nor dogs is bound here
end
-- neither cats nor dogs is bound here
```

Fallthrough was included in and was a major motivator for a previous version of this RFC. It was decided against since in other languages, it often leads to unexpected behavior, possible footguns, and is mostly useful for error catching. Additionally, as proposed in the previous RFC, `if local` fallthrough could easily result in hard-to-understand control flow in Luau that could increase the cognitive complexity of code in the language for little benefit.

As an edge case, locals may be reassigned within the `in` condition. In this case, the conditional branch executes because x is initially not `nil`, allowing the `in` condition to evaluate, reassign `x` to `nil`, and return `true`:

```luau
if local x = 3 in (function() x = nil; return true end)() then
    print(x) -- nil
end
```

### `while local loops`

`while local` statements are defined as `while` loops with one or more `local` bindings. Similarly to `if local`s, `local` bindings may be declared after the `while` keyword, may contain one `in` conditional clause, and must follow with the `do` keyword.

```luau
while local identifier = expr() do
end
-- or
while local identifier = expr() in cond() do
end
```

Local bindings are evaluated once, before the first iteration, and `in` conditions are evaluated once every iteration. If an `in` clause is not specified, the `while local` treats the condition as a `while true` loop and will continue iterating indefinitely or until broken with `break`. Unlike bindings in `if local` statements, bindings in `while local` loops may be initialized to `nil` before their first iteration. Although this seems counterintuitive, it makes sense because the usecase for `if local`s (nilchecking) differs from `while local`s. Additionally, `while` loops don't often encounter `nil` sentinel values unlike `if local` statements, which mostly operate on nilchecking.

## Drawbacks

Why should we *not* do this?

-- TODO

## Alternatives

What other designs have been considered? What is the impact of not doing this?

-- TODO

`if local` *expressions* are not formally included in this RFC due to implementation difficulty, however are included as a future proposal and would follow extremely similar syntax to `if local` statements:

```luau
local humanoid: Humanoid? = if local character = player.Character
    then character:FindFirstChildOfClass("Humanoid")
    else nil
```

For `if local` *expressions* to be possible, Luau would need bindings-in-expressions support, which would require a compiler rewrite.
