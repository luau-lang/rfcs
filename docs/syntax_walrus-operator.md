# Syntax-Assignment-Expression: Walrus Operator (:=)

## Summary

This RFC proposes adding the walrus operator (`:=`) to Luau, allowing for assignment expressions that both assign values to variables and return those values in a single expression, allowing for minified code in scenarios where a value needs to be both stored and immediately used.

## Motivation
Currently in Luau, assignment statements using `=` do not return a value, requiring separate lines for assignment and usage in conditional expressions, function calls, and other contexts.
The walrus operator would allow assignments to be embedded within other expressions, allowing for shorter and readable code.

Some common use cases include:

1. Conditional assignment and testing in a single expression
2. Assigning and using values in loop conditions
3. Reducing repetition and variable scope leakage

## Design
The walrus operator `:=` introduces assignment expressions to Luau. Unlike the standard assignment operator `=` which is used in statements, the walrus operator can be used within expressions
and returns the assigned value.

### Syntax
```
<variable> := <expression>
```

The expression evaluates to the value assigned to the variable.

### Examples

```lua
if local result := someFunction() then
    processResult(result)
end
```

```lua
while local line := readNextLine() do
    processLine(line)
end
```

```lua
local config = {
    value = local result := expensiveFunction(),
    modified = result + 10
}
```

```lua
return validateAndTransform(local parsed := parseData(input))
```

Variables created with the walrus operator in a control structure (if, while, for) are scoped to the block of that control structure. When used in expressions outside control structures, the variable is scoped to the enclosing block.

## Drawbacks
1. Would increase syntax complexity
2. Potential readability issues with some complex one-liners
3. Divergence from standard Lua
4. Generates a different bytecode instruction

## Alternatives
1. **Do nothing** and instead continue requiring separate assignment statements and expressions.
2. **First-class functions:** Encourage higher-order functions for some patterns.
3. **Local Declaration Expressions:** Extend the semantics of the existing local declarations.
4. **Multi-statement Expressions:** Allow for blocks to be used as expressions.

Not implementing this feature would mean Luau code continues to require separate statements for assignment and value usage, leading to wider variable scopes than necessary.
