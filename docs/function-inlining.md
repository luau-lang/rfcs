# No support for user inlining (yet)

## Summary
<!-- https://luau-lang.org/performance#function-inlining-and-loop-unrolling -->
Luau's bytecode compiler (or VM?) is smart about inlining functions. We would rather focus on improving this automation than risk the chance of its misuse.

## Motivation

**context**: function inlining replaces function calls with the function's implementation to reduce overhead costs.

Whether the goal is to improve instruction locality or perform context specific optimizations, <!-- force the compiler to optimize the function body in specific contexts --> function inlining can be a great resource for developers. Thankfully with the `-02` switch, Luau is able to automatically decide if code deserves to be inlined and optimize the code accordingly. To the best of our knowledge, exposing this to users directly could do more harm than good. Not only do these use cases only cater to a tiny subset of the language's users, it exposes our bytecode compiler to FILL when the larger subset who may not have a working understanding of the compiler adopts it.

```luau
@inline
local function sum(lst: {[number]})
    if #lst == 0 then
        return 0
    else
        return lst[1] + sum(lst) -- bad rec
    end
end

local s = sum({1, 2, 3})
```

Consider this "inlined" recursive call, it attempts to sum up numbers in a list but it never terminates as `sum` is recursively called on `lst` which does not shrink. If Luau attempted to inline such a call, it would unroll this and one or two things could happen. If we did have an unroll limit then we would attempt our unrolling up until that point and end up making the program larger than it needs to be or thrashing our stack. The stack will grow beyond the system's stack limit and could potentially eat up resources meant for other processes.
<!-- one or two also sounds funny, change -->

While this may not be the most repe

## Design
None (yet).
<!-- additionally, we could just take the inline hints and ignore if it does not meet some criteria (unroll limit) -->

## Drawbacks
None (yet).

## Alternatives
Prior to now, the team has considered some designs to make function inlining work effectively in Luau and is still hopeful that this may be possible sometime in the future. Till then, we suggest opening github issues with code samples of legitimate cases that we can use to further improve Luau's automatic (function inlining and unrolling) optimizations.

