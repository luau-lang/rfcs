# Labeled break and continue

## Summary

Introduce a labeled form of loop, and extend `break` and `continue` to optionally reference a label. The label is written as a colon-suffix on the loop's `do` keyword — `for i = 1, 10 do: outer` — making it structurally bound to the loop it names. This allows structured non-local exits from nested loops without workarounds such as flag variables or immediately-invoked function expressions.

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

A label is introduced by appending `:` and a name directly to the `do` keyword that opens a loop body:

```
forstat  ::= 'for' ... 'do' ':' Name block 'end'
whilestat ::= 'while' exp 'do' ':' Name block 'end'
repeatstat ::= 'repeat' ':' Name block 'until' exp
```

Because `repeat` has no `do` keyword, its label is placed after `repeat` itself using the same colon-suffix form, keeping the pattern consistent: the label always immediately follows the opening keyword of the block it names.

The labeled form is distinct from the unlabeled form — the label is mandatory once the `:` is written. There is no optional label on an otherwise-unlabeled `do`.

Because the label is syntactically part of the `do` or `repeat` keyword, it is structurally impossible to attach a label to any other kind of statement. An `if` block, a `local` declaration, a function call — none of these contain `do` or `repeat` in opening position, so the `do: name` form simply cannot appear on them. This is not a runtime or semantic restriction that must be checked and reported as an error; it is enforced by the grammar itself. Prefix label syntaxes (such as `::name:: stat`) do not carry this property: `::name:: if ...` or `::name:: local x = 1` would be grammatically well-formed and would require an explicit semantic rejection pass to rule out. The `do:` form makes the constraint self-documenting — a reader seeing `do: label` immediately understands that labels attach to blocks, not to arbitrary statements.

`break` and `continue` are extended to accept an optional bare label name:

```
breakstat    ::= break [Name]
continuestat ::= continue [Name]
```

When no label is given, `break` and `continue` behave exactly as today: they refer to the innermost enclosing loop. When a label is given, `break label` exits the labeled statement and `continue label` skips to the next iteration of the labeled loop.

### Scoping rules

A label is in scope from the `do:` (or `repeat:`) token to the end of the enclosing block. It is visible in all nested blocks and functions — but only for the purpose of `break`/`continue`; a label cannot be referenced in any other context, and it is never captured as an upvalue.

Label names must be unique within their immediately enclosing block. Redeclaring a label name at the same scope level is a compile-time error. Shadowing an outer label with an inner label of the same name is permitted, following the same rules as variable shadowing.

It is a compile-time error to `break` or `continue` a label that is not in scope.

### Grammar disambiguation

`do` is a keyword with no existing interpretation that allows `:` to follow it, so `do:` is entirely unambiguous at the parser level. No lookahead is required — the parser commits to the labeled form as soon as it sees the `:` token immediately after `do` (or `repeat`).

Label references in `break label` and `continue label` use a bare identifier. Since `break` and `continue` are keywords that currently take no arguments, any identifier following them on the same line is unambiguously a label reference.

### Interaction with `continue`

`continue` is already a contextual keyword in Luau. The labeled form `continue Name` is parsed the same way: `continue` followed by an identifier on the same line. Since `continue` only appears inside loop bodies and cannot start a function call, there is no ambiguity with expression statements.

### Type checking

Labels have no direct type implications. The type checker treats labeled `break` the same as an unconditional `break` for the purposes of control-flow analysis: it is a definite exit from the labeled statement. Labeled `continue` is treated as a definite jump to the loop condition of the labeled statement.

Type narrowing inside a loop is not affected: the compiler already conservatively widens types at loop back-edges, and labeled `continue` is just another back-edge to the labeled loop's header.

### Examples

**Breaking out of a nested loop:**

```lua
for _, chunk in chunks do: search
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
for _, row in grid do: outer
    for _, cell in row do
        if cell.blocked then
            continue outer  -- skip this entire row
        end
        process(cell)
    end
end
```

**Multiple label levels:**

```lua
for _, page in document.pages do: pages
    for _, section in page.sections do: sections
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

**Labeled `while` and `repeat`:**

```lua
while not queue:empty() do: processing
    local item = queue:pop()
    for _, dep in item.dependencies do
        if dep:failed() then
            continue processing  -- skip this item, try next
        end
    end
    item:process()
end

repeat: retries
    local ok, err = attempt()
    if err == "fatal" then break retries end
until ok or retries_exceeded()
```

**Label shadowing:**

```lua
for i = 1, 10 do: outer
    for j = 1, 10 do: outer  -- OK: shadows outer label in this scope
        if j == 5 then break outer end  -- breaks the inner 'outer' label
    end
end
```

### Bytecode

`break label` compiles to the same `JUMP` instruction as an unlabeled `break`, targeting the instruction immediately following the labeled statement. `continue label` compiles to the same `JUMP` as an unlabeled `continue`, targeting the loop condition of the labeled statement. No new bytecode instructions are required.

## Drawbacks

**`repeat` asymmetry.** All other labeled constructs attach the label to `do`, which is the shared block-opening keyword. `repeat` has no `do`, so its label sits on `repeat` itself (`repeat: name`). This is a minor inconsistency programmers must learn.

**Contextual keyword extension.** `continue` is already a contextual keyword. `break label` and `continue label` require the parser to recognize that the identifier following `break`/`continue` on the same line is a label reference. This is straightforward but must be carefully specified to avoid edge cases at line boundaries.

**Increased language surface area.** Labels are a feature programmers must learn. Deeply nested loops with non-obvious label names can reduce readability. Poor use of labeled breaks can produce code nearly as hard to follow as `goto`.

## Alternatives

**`goto`.** Lua 5.2 introduced `goto` with `::label::` jump targets. Luau does not include `goto` because it is difficult to sandbox, complicates control-flow analysis, and interacts poorly with type narrowing and upvalue capture. The Luau team has explicitly discussed and decided against extending Luau with arbitrary `goto` — this is a settled position, not an open question. Labeled `break`/`continue` achieves the primary practical use case of `goto` — non-local loop exit — without those problems, because the target is always an enclosing structured statement.

**`::label:: loopstat` prefix syntax.** Placing the label before the loop as `::name:: for ...` mirrors Lua 5.2's `goto` label token form and has no disambiguation issues. The `do: name` form was preferred for two reasons. First, the label sits at the natural reading boundary — the `do` keyword where the loop body opens — rather than floating before the loop header. Second, and more importantly, the prefix form does not grammatically restrict which statements can be labeled: `::name:: if ...` or `::name:: local x = 1` are well-formed under a prefix grammar and must be rejected by a separate semantic pass, whereas `do: name` is only grammatically valid on constructs that contain `do` or `repeat`, making the restriction self-enforcing. Sharing `::name::` with Lua's `goto` syntax would have been a cross-compatibility benefit (Lua code using `goto` for loop exit can be ported to Luau by moving the label from a standalone `::name::` target to the `do:` position), but the structural clarity of the `do:` form was judged to outweigh that.

**`Name ':' loopstat` prefix syntax.** Using a single trailing colon (`outer: for ...`) matches JavaScript, Java, and Kotlin precedent and is concise. It was not chosen because in Luau, `Name ':'` at statement level is already the start of a method call (`foo:bar()`), requiring lookahead disambiguation. The `do: name` form has no such conflict.

**`@label` or `#label` sigil syntax.** A sigil prefix (e.g., `@outer: for ...`, `break @outer`) would make label references visually distinct from identifiers. However, `@` conflicts directly with Luau's attribute syntax (`@native`, `@inline`), which also begins with `@` followed by an identifier at statement or declaration position. A statement beginning with `@name` is therefore ambiguous between an attribute annotation and a label declaration without deep lookahead: the parser cannot commit until it sees what follows the identifier — a `:`  for a label, or a function/variable declaration for an attribute — potentially spanning many tokens. This would complicate the parser significantly and create a confusing authoring experience where `@name for` means something entirely different from `@name function`.
`#label` is ambugiuous with `<#> <ident>` syntax there `#` is the length operator.

**IIFE workaround.** The status quo. Viable but carries allocation overhead, obscures intent, complicates type checking, and produces idiomatic code that exists only because of a language limitation rather than expressing a meaningful abstraction.

**Flag variables.** The other common workaround. No overhead, but requires additional mutable state and conditional checks at every loop level between the exit point and the target, which adds noise proportional to nesting depth.
