# Unordered `table.remove()`

## Summary

Add an unordered remove method to the table library.

## Motivation

For arrays that are known to be unsorted, using `table.remove()` to remove indicies is unperformant, as `table.remove()` takes O(n) time to complete. This is a gross use of resources, especially if this behavior is not required by the user. Thus, it's logical to conclude that an analogous method that performs an *unordered remove* would be the clear next step.

## Design

The naming of the new method is proposed as `uremove`. However, for clarity reasons, this may not be the best name choice as `uremove` types very similar to `remove`. The implementation of the method could be as such:
```lua
function table.uremove(t: {any}, index: number)
	t[index], t[#t] = t[#t], nil
end
```
Valid argument checking is optional.

## Drawbacks

Possibly none, apart from an extra method in the table library.

## Alternatives

Users can define the function themselves.
