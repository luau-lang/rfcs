# Labeled break and continue

## Summary

Introduce optional labels on loop statements (`for`, `while`, `repeat`) and `do` blocks, and extend `break` and `continue` to optionally reference a label. This allows structured non-local exits from nested loops without workarounds such as flag variables or immediately-invoked function expressions.

## Motivation

It is common in Luau code to iterate over nested data structures and want to abort or skip iteration at an outer level once some condition is met. Today, Luau provides no direct mechanism for this. The available workarounds each have significant problems.

**Flag variables** add noise and can obscure intent:

```lua
local found = false
for _, chunk in chunks do
    for _, entity in chunk.entities do
        if entity:matches(target) then
            found = true
            break
        end
    end
    if found then break end
end
```

**Immediately-invoked function expressions (IIFEs)** are commonly used in Luau codebases:

```lua
local result = (function()
    for _, chunk in chunks do
        for _, entity in chunk.entities do
            if entity:matches(target) then
                return entity
            end
        end
    end
    return nil
end)()
```

This pattern has several drawbacks:

- It reuses `return` with different semantics than the enclosing function, which can confuse readers.
- Luau's type checker must reason through the IIFE boundary, which complicates type narrowing and return type inference.
- It is stylistically alien — the construct exists solely to work around a language limitation rather than to express intent.
- If compiled without optimization(or inlining doesn't happen), it introduces a closure allocation and call overhead. The IIFE takes zero arguments and captures locals as upvalues. Luau's inlining cost model deprioritizes such functions, so the call is unlikely to be inlined. Even when inlining does occur, the lambda's prototype remains in the bytecode — occupying space and contributing to parse time.

Labeled `break` and `continue` address all of these problems with a straightforward, well-precedented mechanism found in Java, Kotlin, JavaScript, Rust, and other languages. It is the natural complement to the `continue` statement already present in Luau.

## Design

### Syntax

A label may be placed immediately before a `for`, `while`, `repeat`, or `do` statement. The label uses double-colon delimiters, matching the label token form used in Lua 5.2:

```
labeledstat ::= '::' Name '::' loopstat
loopstat    ::= forstat | whilestat | repeatstat | dostat
```

The label name is a plain identifier and follows normal Luau identifier rules. Label names share no namespace with local variables, upvalues, or global variables; a label named `i` does not shadow or conflict with a variable named `i`.

`break` and `continue` are extended to accept an optional label name:

```
breakstat    ::= break [Name]
continuestat ::= continue [Name]
```

Note that the label *reference* in `break` and `continue` uses a bare name — only the *declaration* uses `::name::`. This keeps the call-site syntax concise while making label declarations visually prominent and syntactically unambiguous.

When no label is given, `break` and `continue` behave exactly as today: they refer to the innermost enclosing loop.

When a label is given, `break label` exits the statement identified by `label`, and `continue label` skips to the next iteration of the loop identified by `label`.

### Scoping rules

A label is in scope from the point of its declaration to the end of the enclosing block. It is visible in all nested blocks and functions — but only for the purpose of `break`/`continue`; a label cannot be referenced in any other context, and it is never captured as an upvalue.

A label that names a `do` block may be targeted by `break` but not `continue`, because `do` blocks do not iterate. Attempting to `continue` a `do` block is a compile-time error.

Label names must be unique within their immediately enclosing block. Redeclaring a label name at the same scope level is a compile-time error. Shadowing an outer label with an inner label of the same name is permitted, following the same rules as variable shadowing.

It is a compile-time error to `break` or `continue` a label that is not in scope.

### Grammar disambiguation

Because label declarations begin with `::`, there is no ambiguity with any existing Luau syntax. The `::` token is not valid at the start of a statement expression, so the parser can commit to the label rule as soon as it sees `::` in statement position.

Label references in `break label` and `continue label` use a bare identifier. Since `break` and `continue` are keywords that currently take no arguments, any identifier following them on the same line is unambiguously a label reference. The parser handles this with a single-token lookahead after `break`/`continue`: if the next token is an identifier, it is consumed as the label name.

### Interaction with `continue`

`continue` is already a contextual keyword in Luau. The labeled form `continue Name` is parsed the same way: `continue` followed by an identifier on the same line. Since `continue` only appears inside loop bodies and cannot start a function call, there is no ambiguity with expression statements.

### Type checking

Labels have no direct type implications. The type checker treats labeled `break` the same as an unconditional `break` for the purposes of control-flow analysis: it is a definite exit from the labeled statement. Labeled `continue` is treated as a definite jump to the loop condition of the labeled statement.

Type narrowing inside a loop is not affected: the compiler already conservatively widens types at loop back-edges, and labeled `continue` is just another back-edge to the labeled loop's header.

### Examples

**Breaking out of a nested loop:**

```lua
::search:: for _, chunk in chunks do
    for _, entity in chunk.entities do
        if entity:matches(target) then
            print("found", entity)
            break search
        end
    end
end
```

**Continuing an outer loop from an inner loop:**

```lua
::outer:: for _, row in grid do
    for _, cell in row do
        if cell.blocked then
            continue outer  -- skip this entire row
        end
        process(cell)
    end
end
```

**Breaking a `do` block to skip a section of code:**

```lua
::validate:: do
    if not config.enabled then break validate end
    if config.value < 0 then
        warn("negative value")
        break validate
    end
    applyConfig(config)
end
```

**Multiple label levels:**

```lua
::pages:: for _, page in document.pages do
    ::sections:: for _, section in page.sections do
        for _, word in section.words do
            if word == stopWord then
                break pages     -- exit all three loops at once
            end
            if word == sectionBreak then
                break sections  -- exit inner two loops
            end
            index(word)
        end
    end
end
```

**Label shadowing:**

```lua
::outer:: for i = 1, 10 do
    ::outer:: for j = 1, 10 do  -- OK: shadows outer label in this scope
        if j == 5 then break outer end  -- breaks the inner 'outer' label
    end
end
```

### Bytecode

`break label` compiles to the same `JUMP` instruction as an unlabeled `break`, targeting the instruction immediately following the labeled statement. `continue label` compiles to the same `JUMP` as an unlabeled `continue`, targeting the loop condition of the labeled statement. No new bytecode instructions are required.

## Drawbacks

**Contextual keyword extension.** `continue` is already a contextual keyword. `break label` and `continue label` require the parser to recognize that the identifier following `break`/`continue` on the same line is a label reference. This is straightforward but must be carefully specified to avoid edge cases at line boundaries.

**Increased language surface area.** Labels are a feature programmers must learn. Deeply nested loops with non-obvious label names can reduce readability. Poor use of labeled breaks can produce code nearly as hard to follow as `goto`.

**Association with `goto`.** The `::name::` token form originates in Lua 5.2's `goto` feature, which Luau deliberately excludes. Using the same delimiter for loop labels may cause confusion about whether `goto` is also supported. This is mitigated by the fact that `::name::` here always precedes a structured statement rather than appearing as a standalone jump target. On the other hand, sharing the label syntax with Lua 5.2+ is an explicit advantage: Lua code that uses `goto` for nested-loop exit can be mechanically ported to Luau by replacing `goto label` with `break label` or `continue label` and moving the `::label::` annotation from a standalone jump target to the front of the corresponding loop, with no change to the token form of the label itself. It also means Lua developers reading Luau code will immediately recognise the `::name::` delimiter, lowering the learning curve.

## Alternatives

**`goto`.** Lua 5.2 introduced `goto` with `::label::` jump targets. Luau does not include `goto` because it is difficult to sandbox, complicates control-flow analysis, and interacts poorly with type narrowing and upvalue capture. The Luau team has explicitly discussed and decided against extending Luau with arbitrary `goto` — this is a settled position, not an open question. Labeled `break`/`continue` achieves the primary practical use case of `goto` — non-local loop exit — without those problems, because the target is always an enclosing structured statement.

**`Name ':' loopstat` syntax.** Using a single trailing colon (`outer: for ...`) matches JavaScript, Java, and Kotlin precedent and is less verbose. It was considered but not chosen because in Luau, `Name ':'` at statement level is already the start of a method call (`foo:bar()`), requiring lookahead disambiguation. The `::name::` form has no such conflict and makes label declarations visually prominent, reducing the chance of misreading a label as a method call.

**`@label` or `#label` sigil syntax.** A sigil prefix (e.g., `@outer: for ...`, `break @outer`) would make label references visually distinct from identifiers. However, `@` conflicts directly with Luau's attribute syntax (`@native`, `@inline`), which also begins with `@` followed by an identifier at statement or declaration position. A statement beginning with `@name` is therefore ambiguous between an attribute annotation and a label declaration without deep lookahead: the parser cannot commit until it sees what follows the identifier — a `:`  for a label, or a function/variable declaration for an attribute — potentially spanning many tokens. This would complicate the parser significantly and create a confusing authoring experience where `@name for` means something entirely different from `@name function`.
`#label` is ambugiuous with `<#> <ident>` syntax there `#` is the length operator.

**IIFE workaround.** The status quo. Viable but carries allocation overhead, obscures intent, complicates type checking, and produces idiomatic code that exists only because of a language limitation rather than expressing a meaningful abstraction.

**Flag variables.** The other common workaround. No overhead, but requires additional mutable state and conditional checks at every loop level between the exit point and the target, which adds noise proportional to nesting depth.
