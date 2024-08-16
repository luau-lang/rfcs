# Constant tau

## Summary

Define a constant value `math.tau` to be the tau constant.

## Motivation

Currently the value of tau is used in trigonomic calculations through `math.pi * 2`.

The implementation of tau will simplify expressing angles in turns or equations where otherwise math.pi * 2 may be used such as:

```
-- angular difference along a sphere
local sphereAngleDiff_pi = (a - b + math.pi / 2) % (math.pi * 2)
local sphereAngleDiff_tau = (a - b + math.pi / 2) % math.tau

-- angular frequency calculation
local frequency_pi = math.pi * 2 * angularFrequency
local frequency_tau = math.tau * angularFrequency

-- random gradient

-- pi form
local r = math.random() * math.pi * 2
-- tau form
local r = math.random() * math.tau

local gradient = vector(math.cos(r), math.sin(r), 0)

-- pi form
local radsPerSec = 3.5 * math.pi * 2 -- 3.5 revolutions per second
-- tau form
local radsPerSec = 3.5 * math.tau -- 3.5 revolutions per second
```

## Design

Define `math.tau` with the value `6.28318530717958647692528676655900577`

## Drawbacks

None
