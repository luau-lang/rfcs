# Feature name

## Summary

Fundemental function that takes in vardiact numbers and returns
the mean.

Averaging a sequence of numbers also called arithmetic average
where the sum is devided by count.

```luau 
type avg = (...:number) -> number
```

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

### @nnullcolumn suggested a better example than averaging strings.

Unlike other averaging methods this is the simplest and most
commonly used to answer one root question:
What is the Mean of this Data.

The examples are limitless especially in the field of statistic
where your job is to use geathered information (data),
and turn it into valuable information like:

- Track users activities and recommend them to other
users with similar activities. To boost social interaction.

- Measure someones skill and rank them based on that mean.
Skipping traditional boring linear ranking methods.

- Calculating the attention spans of users
take the mean of that and now you have power over the exit button.
You can create hooks to boost the attention spans,
just when their attention is about to give out (in
a probabilistic scale)

### @nnullcolumn suggested to address potential accuracy gains.

We know that developers will love the globalization of `math.avg`
and the native implementation will overrun any user-defined alternatives,
but what about accuracy gains?

### Pairwise (Cascade) Summation (Recommended Best Balance)

Instead of adding the numbers linearly (a+b+c+d) we could split array in half and add the pairs ([a+b]+[c+d]).

### Standard Linear Accumulation (Performance Over Precision)

VM simply uses a standard double accumulator loop

### Kahan/Neumaier Compensated Summation (Highest Precision)

An algoritm that tracks the rounding errors in a seperate variable and introduces them
into the next iteration. This has a high precision for cost of preformance.

My goal was to globalize `math.avg`; to avoid user-defined alternatives but
taking those facts into consideration before implementation could help in the long run.

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

### @SPY asked: What if the function gets called without arguments?

```luau
math.avg()
```

### @razvanpacku suggested for the function to throw an exception to match the other math vardiac functions.

```luau
math.avg() --> error argument #1 expects number, got nil!

local numbers = {}

math.avg(table.unpack(numbers)) --> error argument #1 expects number, got nil!
```

This prevent silent failures and forces the developer to handle it.

### @nnullcolumn suggested that the function should return 0

```luau
math.avg() --> 0

local numbers = {}

math.avg(table.unpack(numbers)) --> 0

-- weakness: 0 is not guaranteed to be caused by the process of the function.

math.avg(-2, -1, 1, 2) --> also returns 0
```

Increases silent failures and ambiguity.
This will also give two meaning to zero.
- Dataset can be empty.
- Dataset is balanced.

This forces the developer to double check which defeates the purpose of a shorthand function.

### How this function should handle none valid numbers (inf, nan)?

```luau
math.avg( math.huge ) --> inf
math.avg( 0 / 0 ) --> nan
```

This is a developer error and should be handled by the developer.

Another aproach is if we ignore none valid numbers and only work with what we have,
which adds to the complexity of the function and will slow it down.

## Drawbacks

This adds another function to the standard library.

This may add requests for further averaging functions 
(like median or mode) to better represent the center of a dataset.

Limitations of Mean averaging:
- Outliner vanurability: A single very high or low number can influence the outcome of the function.
- Asymmetric datasets: Are data sets who are not equally distributed and can shift the mean to the tail.
- Hides spread: turns data catagories and variations into one matric number.
- Non existing: Mean is pure calculation and the result may not exist inside of the dataset.

## Alternatives

Not implementing this will result in user-defined alternatives.

Another option is if [math.sum](https://github.com/HarmosCreations/rfcs_harmos/blob/master/docs/feature_math.sum.md) is implemented
which removes a need for this function (ignoring native implemention benefits).

```lua
print( math.sum( string.byte("123", 1, 3) ) / 3 ) --> 50
```
