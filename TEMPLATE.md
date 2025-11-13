# Table Pre-Allocation Improvements

## Summary
The current table creation process doesn't allow explicit preallocation for dictionaries or mixed tables.
This proposal introduces a table.allocate function for mixed/dictionary tables and allows static calls to table.create to be compiled directly to NEWTABLE.

## Motivation

As of now you can't allocate dictionary in any way and so is mixed table even thought NEWTABLE can allocate both, this proposal would solve this with introduction of dedicated function and bytecode compilation related optimizations
`table.create(100)` could be compiled into `NEWTABLE R0 100 0` instead of being a function call if value passed is static

Exposing both through direct compilation or new API improves performance in systems that frequently construct tables with known sizes (e.g. serialization, ECS, parsing, etc.).

## Design


Function table.allocate would be designed specifically for mixed or dictionary tables unlike table.create that is optimized for arrays, similar to NEWTABLE opcode but dynamic, perhabs possibility for this function to be introduces as a fast call or a separate opcode that accepts registers to allocate dynamic table size.
structure:
```lua
function table.allocate(array:number?,dictionary:number?):{[any]:any}
```
Similar to my proposal with table.create this function could be compiled right into `NEWTABLE` opcode if passed variables are static.
```lua
-- existing behavior
local t = table.create(100)
-- proposed compiled form:
--NEWTABLE R0 100 0

-- new API
local mixed = table.allocate(16, 32)
-- proposed compiled form (if both args are constants):
-- NEWTABLE R1 16 32

--And for dynamic cases:

local t = table.allocate(a, b)
-- emitted as a function call
```

