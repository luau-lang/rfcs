# math.truncate

## Summary

A method that returns the integer part of a number by removing any fractional digits. Along with specifying how many digits should be kept after the decimal point.

## Motivation

``math.floor`` does not give the same effect as, for instance ``Math.trunc()`` from JavaScript. But ``math.floor`` and ``math.ceil`` Can be used together.
But JavaScript only truncates, it doesn't feature 

For instance:
```lua
print( math.floor(-0.005) )
-- Returns: -1

print( math.ceil(-0.005) )
-- Returns: -0
```

```js
Math.trunc(-0.005)
// Returns: -0
```


## Design

``math.truncate(num, idp)``
- ``num``: Number to truncate
- ``idp``: _(Optional)_, How many digits to keep after the decimal point


```lua
math.truncate(-0.005)
-- Expected output: -0

math.truncate(0.005)
-- Expected output: 0

math.truncate(1.23456789, 4)
-- Expected output: 1.2345

math.truncate(-math.huge)
-- Expected output: -inf
```


## Drawbacks

- Should ``idp`` be a thing? Since one can just do this _(JavaScript example)_

```js
function trunc_with_idp(num, idp) {
    let mul = Math.pow(10, (idp || 0))
    return Math.trunc(num * mul) / mul
}
trunc_with_idp(1.2345, 4)
```

_or see alternative below_


## Alternatives

From [here](https://github.com/Facepunch/garrysmod/blob/8e96e31fc9ab9de053edc0513f9d7a6f3e4be1e1/garrysmod/lua/includes/extensions/math.lua#L168-L171)
```lua
function math.Truncate( num, idp )
	local mult = 10 ^ ( idp or 0 )
	return ( num < 0 and math.ceil or math.floor )( num * mult ) / mult
end
```
