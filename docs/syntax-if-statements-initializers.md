# Initializers in if statements

# Summary

Introduce an initializer expression that declares and initializes variables in if statements.

# Motivation

Initializers can improve code clarity and reduce scope pollution by allowing developers to declare variables in the conditions of if statements. ⁤⁤ Declaring variables at the point of use in if statements for the scope of the if statement's block simplifies code, leading to better readability and understanding of program logic. By limiting the scope of variables to the condition's pertaining block, the risk of unintended variable reuse and naming conflicts is reduced. ⁤

The reduced scope pollution improves register space in extreme cases (or auto-generated code) where developers have many variables defined and have to work around register limits by reducing the register size. In some scenarios, especially on Roblox, initializing variables within if statements can lead to improved performance and reduced complexity by avoiding unnecessary calls by developers. ⁤ A common paradigm used by Roblox developers is to use `Instance:FindFirstChild` in their condition and then access it afterwards instead of keeping an existing variable around in an existing scope, polluting it.

# Design

If statements with initializers must match (following the Luau grammar) `'if' 'local' bindinglist ['=' explist] 'then'` and `'local' bindinglist ['=' explist] where exp 'then'` syntax. Parenthesis are also allowed around the initializer for consistency with other expressions. The variables declared by the initializer are only available to the block of that condition and will be undefined to the `elseif` conditions and blocks as well as the `else` block; any code after the if statement won't have the variables defined either.

In the former case, the value of the first declared variable will be checked.

Example:

```lua
local function foo()
    return true
end

if local b = foo() then
   print(b)
else
   print("false")
end
```

`Output: true`

In the latter case, the `exp` condition is checked rather than the initializer.

Example:

```lua
local function foo()
    return true
end

if local b = foo() where b == false then
   print(b)
else
   print("true")
end
```

`Output: true`

When declaring multiple values inside of a condition, only the first condition will be evaluated.

Example:

```lua
local function foo()
    return true, false
end

if local a,b = foo() then
    print'Hello World, from Luau!'
end
```

`Output: Hello World, from Luau!`

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

While Luau is a verbose language that uses keywords for the majority of its syntax, another approach is using semicolons as a separator. This can work well because statements can use semicolons as a separator, which will retain consistency with the language.

```lua
if local a, b = foo(); b > a then
  print'Hello World, from Luau!'
end
```
