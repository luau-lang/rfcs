# math.avg

## Summary

Fundemental function that takes in vardiact numbers and returns
the avarage of them all.

## Motivation

Averaging numbers is a common operation in game development
and to avoid user-defined repetition code like this:

```lua

local total= 0

for _, num in {string.byte("123", 1, 3)} do
 total += num
end

local avg = total/#{string.byte("123", 1, 3)}

print(avg) --> 50

```

## Design

The speciality about this function is the arguments,
it only takes one vardiac argument and automatically
calculates the average of it.

```lua

function math.avg(...: number) : number
 -- without using math.sum function
 local total = 0

 for _, num in {...} do
  total += num
 end

 return total/#{...}
end

print(math.avg(1,2,4,6)) --> 3.25
print(math.avg(table.unpack({5, 5, 3, 2})) --> 3.75
print(math.avg(string.byte("123", 1, 3)) --> 50

```

## Drawbacks

This adds another function to the standard library.

## Alternatives

Not implementing this will result in user-defined alternatives and This would keep the same problems as before.

Another option is if [math.sum](https://github.com/HarmosCreations/rfcs_harmos/blob/master/docs/feature_math.sum.md) is implemented
that would simplify the user-defined functions.

```lua
local avg = math.sum(string.byte("123", 1, 3))/#{string.byte("123", 1, 3)}
print(avg) --> 50
```
