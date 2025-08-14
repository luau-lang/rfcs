# Table Dictionary Length

## Summary

A built-in function to get the length of the dictionary part of a table.

## Motivation

An O(1) method to count the number of elements in the dictionary part of the table by lifting the information directly from the underlying data structure.  Without this, it's only possible to get the count by premeditated management of a count variable or by outright counting on the spot.

## Design

Add a function `table.dictlen(t)` to the standard library which returns the size of the dictionary part of the given table.

## Drawbacks

If Luau's C++ Lua table implementation has no concept of its current length and the algorithmic complexity of the operation is no better than outright counting, then there's no reason to add this.

## Alternatives

Maintaining a count variable manually via proxy functions.  The implementation below has some caveats: it only counts correctly when inserting an element that hasn't already been inserted or when removing an element that hasn't already been removed.
```Lua
local t={}
local t_count=0
local function t_insert(i,v)
	t[i]=v
	t_count+=1
end
local function t_remove(i)
	t[i]=nil
	t_count-=1
end
```
