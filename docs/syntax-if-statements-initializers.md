# Initializers in if statements

# Summary

Introduce an initializer expression that declares and initializes variables in if statements.

# Motivation

Initializers can improve code clarity and reduce scope pollution by allowing developers to declare variables in the conditions of if statements. ⁤⁤ Declaring variables at the point of use in if statements for the scope of the if statement's blocks simplifies code, leading to better readability and understanding of program logic. By limiting the scope of variables to the if statement's blocks, the risk of unintended variable reuse and naming conflicts is reduced. ⁤

The reduced scope pollution improves register space in extreme cases (or auto-generated code) where developers have many variables defined and have to work around register limits by reducing the register size. In some scenarios, especially on Roblox, initializing variables within if statements can lead to improved performance and reduced complexity by avoiding unnecessary calls by developers. ⁤ A common paradigm used by Roblox developers is to use `Instance:FindFirstChild` in their condition and then access it afterwards instead of keeping an existing variable around in the new scope, polluting the existing scope.

# Design

If statements with initializers must match (following the Luau grammar) `'if' 'local' bindinglist ['=' explist] 'then'` and `'local' bindinglist ['=' explist] where exp 'then'` syntax. The variables declared by an initializer are only available to the if statement's blocks; any code after the if statement won't have the variables defined.

In the former case, the value of the first declared variable will be checked.

Example:

```lua
local function foo()
    return true
end

if local b = foo() then
   print(b, "truthy block")
else
   print(b, "falsy block")
end
```

`Output: true	truthy block`

In the latter case, the `exp` condition is checked rather than the initializer.

Example:

```lua
local function foo()
    return true
end

if local b = foo() where b == false then
   print(b, "truthy block")
else
   print(b, "falsy block")
end
```

`Output: true	falsy block`

When declaring multiple values inside of an initializer, the `where` clause is required.

Example:

```lua
local function foo()
    return true, false
end

if local a,b = foo() where a and b then
else
    print'Hello World, from Luau!'
end
```

`Output: Hello World, from Luau!`

If statement initializers are also allowed in `elseif` conditions.

Example:

```lua
local a = false
local function foo()
    local b = a
    a = true
    return b
end

if local a = foo() then
elseif local b = foo() then
    print(b)
end
```

`Output: true`

# Drawbacks

Parser recovery may be more fragile due to the `local` keyword.

Initializers increase the complexity of the language syntax and may obscure control flow in complex conditions.

# Alternatives

A different keyword or token can be used in place of `where`.

Introducing a new contextual keyword can introduce complexity in the parser, and a different existing keyword like `in` or `do` could be used instead.

```lua
if local a, b = foo() in b > a then
  print'Hello World, from Luau!'
end
```

While Luau is a verbose language that uses keywords for the majority of its syntax, another approach is using semicolons as a separator. This can work well because statements can use semicolons as a separator, which will retain consistency with the language. The same can be said for the comma, which would be consistent with for loop syntax.

```lua
if local a, b = foo(); b > a then
  print'Hello World, from Luau!'
end
```
