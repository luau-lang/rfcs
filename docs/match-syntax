# Feature name

## Summary

This RFC proposes new syntax which allows the developer to write match statements/expressions.

Syntax:
```luau
local exp = player.exp
local rank = match exp
    1 or 2 => "Noob",
    3 # 10 => "Amateur",
    11 # 20 => "Normal",
    21 # 30 => "Experienced",
    31 # 40 => "Pro",
    45 => "The 45th point",
    46 # 50 => "Master",
    else if exp <= 60 => "Super Experienced",
    else => do
        print("unknown rank")

        return "unknown"
    end,
end

print(`Your rank is: {rank}`)
```

## Motivation

This syntax elimiates long and partially unreadable and/or chains and if/else chains while providing the same functionality and being more modern.

## Design

The initial match expression starts with the match keyword (contextual and can only be followed by an identifier) and then with the value to match.

Every match arm is followed by a pattern and then either an expression (which gets returned) _or_ a `do` block with executes and returns.
The only exception is the `else` match arm which is a wildcard and only runs if the value was not matchable to any pattern and always has to be declared last.

There are multiple patterns proposed here:
- Literal pattern: Consists of any number / string / nil / table literal
- Or pattern: Consists of two **patterns** seperated by the `or` keyword
- Range pattern: Consists of two **literals** seperated with a hashtag (or the `until` keyword) and indicates a range of numbers to match
- Guard pattern: Consists of a pattern followed by the `if` keyword and an expression (condition does not get evaluated until the inital pattern was matched)

The `else` arm can be written multiple times if it's a guard pattern and the same guard pattern is not used in any other `else` arm.

The compilation can be done in two different ways:

1. Desugaring into an `if` statement
- Initial match gets turned into an `if` statement
- Every arm gets turned into an elseif (except for the initial arm)
- Output value gets stored in a register slot

2. Creating custom instructions and control flow
- Match statement opens in the bytecode with a `LOP_MATCH` opcode (match value gets stored in slot A)
- Control flow is directed using `LOP_JUMPIF*` and `LOP_JUMPIFNOT*` opcodes

The type of the output value will be either an intersection of every type returned by every expression / block or just be the return type if all of them only return one type.

Duplicate patterns or invalid ranges **will** throw a compiler error.

## Drawbacks

Adding this syntax would introduce a lot of complications in the parser and make typecheck more difficult.

## Alternatives

The most obvious alternative is to write it as an if statement with elseifs as conditions. This is very much doable and would semantically be equal to the match.
The other alternative is bundling it together as a long and/or chain where you can match for patterns.

## Prior Art

The `match` concept is part of FP (Functional Programming) and has been implemented in a vast majority of languages. A prime example of this is Rust:

```rs
let rank = match exp {
    1 | 2 => "Noob",
    3..10 => "Amateur",
    11..20 => "Normal",
    21..30 => "Experienced",
    31..40 => "Pro",
    45 => "The 45th point",
    46..50 => "Master",
    _ if exp <= 60 => "Super Experienced",
    _ => {
        println!("Unknown rank");

        "unknown"
    }
};
```

The design is very similar to the proposed one.
