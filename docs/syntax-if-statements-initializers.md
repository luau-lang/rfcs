# Initializers in if statements

# Summary

Introduce an initializer expression that declares and initializes variables in if statements.

# Motivation

Initializers can improve code clarity and reduce scope pollution by allowing developers to declare variables in the conditions of if statements. ⁤⁤ Declaring variables at the point of use in if statements for the scope of the if statement's blocks simplifies code, leading to better readability and understanding of program logic. By limiting the scope of variables to the if statement's blocks, the risk of unintended variable reuse and naming conflicts is reduced. ⁤

The reduced scope pollution improves register space in extreme cases (or auto-generated code) where developers have many variables defined and have to work around register limits by reducing the register size. In some scenarios, especially on Roblox, initializing variables within if statements can lead to improved performance and reduced complexity by avoiding unnecessary calls by developers. ⁤ A common paradigm used by Roblox developers is to use `Instance:FindFirstChild` in their condition and then access it afterwards instead of keeping an existing variable around in the new scope, polluting the existing scope. 

Another benefit provided by initializers is being able to compact verbose guard clauses into a singular if statement.   

Example:

```lua
function PlayerControl:GetEntities()
    if self.RemovedSelf then
        return self.Instances
    end

    local entity = self:TryGetEntity()
    if entity == nil then
        return self.Instances
    end

    local index = table.find(self.Instances, entity.Root)
    if index == nil then
        return self.Instances
    end

    table.remove(self.Instances, index)
    self.RemovedSelf = true

    return self.Instances
end
```

```lua
function PlayerControl:GetEntities()
    if self.RemovedSelf then
    elseif local entity = self:TryGetEntity() in entity == nil then
    elseif local index = table.find(self.Instances, entity.Root) then
        table.remove(self.Instances, index)
        self.RemovedSelf = true
    end

    return self.Instances
end
```

# Design

If statements with initializers must match the below grammar. The variables declared by an initializer are only available to the if statement's blocks; any code after the if statement won't have the variables defined.

```diff
  stat = varlist '=' explist |
      ...
      'repeat' block 'until' exp |
-     'if' exp 'then' block {'elseif' exp 'then' block} ['else' block] 'end' |
+     'if' cond 'then' block {'elseif' cond 'then' block} ['else' block] 'end' |
      'for' binding '=' exp ',' exp [',' exp] 'do' block 'end' |
      ...
+ cond = 'local' binding '=' exp ['in' exp] |
+     'local' bindinglist '=' explist 'in' exp |
+     exp
```

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

if local b = foo() in b == false then
   print(b, "truthy block")
else
   print(b, "falsy block")
end
```

`Output: true	falsy block`

When declaring multiple values inside of an initializer, the `in` clause is required.

Example:

```lua
local function foo()
    return true, false
end

if local a, b = foo() in a and b then
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

A different keyword or token can be used in place of `in`.

Rather than use `in`, we could introduce a new contextual keyword, or use a different existing keyword like `do`.

```lua
if local a, b = foo() do b > a then
  print'Hello World, from Luau!'
end
```

While Luau is a verbose language that uses keywords for the majority of its syntax, another approach is using semicolons as a separator. This can work well because statements can use semicolons as a separator, which will retain consistency with the language. The same can be said for the comma, which would be consistent with for loop syntax.

```lua
if local a, b = foo(); b > a then
  print'Hello World, from Luau!'
end
```
