# Dedicated Increment/Decrement Opcodes

## Summary

This proposal introduces dedicated VM opcodes for increment and decrement operations on local variables when the compiler can statically prove they are safe and side-effect free:

- `INC` `Rn`
- `DEC` `Rn`

These opcodes are emitted only when the compiler detects a guaranteed pattern such as:
```luau
x = x + 1
x = x - 1
x += 1
x -= 1
```

## Motivation

Increment and decrement operations are among the most common arithmetic patterns in hot execution paths:

- numeric loops
- array traversal
- ECS systems
- simulation updates
- physics integration
- scheduler counters
- tick systems

Currently, these patterns lower into multiple VM instructions even in the simplest possible case.

Example:
```lua
i = i + 1
```

typically lowers into something conceptually similar to:
```luau
LOADN   R1 1
ADD     R0 R0 R1
```
or:
```luau
ADDK    R0 R0 K0
```
depending on compiler optimization path.

While `ADDK` and `SUBK` reduce constant-loading overhead, they still represent a full arithmetic instruction path rather than a dedicated mutation operation.

Example:
```luau
i = i + 1
```
Conceptually lowers into:
```luau
ADDK R0 K0
```
This still requires:

- constant operand decoding
- generic arithmetic execution path
- multiple operand reads

However, increment/decrement by one is a uniquely common operation that can be represented more efficiently as:
```luau
INC R0
```
This reduces:

- instruction complexity
- operand decoding
- VM overhead
- hot loop instruction footprint

Dedicated opcodes also allow the VM dispatch loop to recognize mutation intent directly rather than rediscovering it from generic arithmetic instructions at runtime.

## Design

Two new VM opcodes are introduced:
```luau
INC Rn ; Rn = Rn + 1
DEC Rn ; Rn = Rn - 1
```
The compiler may emit these opcodes only when all of the following conditions are true:

- target is a number
- operation is provably equivalent to increment/decrement by exactly 1
- no observable side effects exist
- no metamethod participation is possible
- no ambiguous upvalue behavior exists

Allowed lowering examples:
```luau
x = x + 1
x = x - 1
```
may lower into:
```luau
INC Rx
DEC Rx
```
instead of:
```luau
ADDK Rn K0
SUBK Rn K0
```
This proposal intentionally does not introduce:

- `++`
- `--`
- new syntax
- intrinsic functions

The feature exists purely as a compiler/VM optimization.

If the compiler cannot fully prove correctness, it must fall back to existing arithmetic opcodes.

## Drawbacks

This proposal introduces additional VM opcodes, increasing VM complexity slightly.

The optimization target is relatively narrow because it only applies to increment/decrement by one.

Some architectures or future VM optimizations may reduce the practical performance difference between INC/DEC and existing ADDK/SUBK instructions.

The optimization may also provide minimal gains in non-hot execution paths.

## Alternatives

Continue using `ADDK` / `SUBK`

This is the current behavior.

While functional, these instructions are generic arithmetic operations and do not represent direct mutation intent.

They still require:

- constant operand handling
- generic arithmetic execution paths
- multiple operand decoding

This proposal exists to provide a more specialized fast path for extremely common mutation patterns.
### Introduce `++` / `--` syntax

This was rejected because:

- `--` conflicts with Luau comments
- introduces parsing ambiguity
- adds new language semantics
- increases syntax complexity
- does not speed up already existing code

This proposal intentionally avoids syntax changes entirely.
