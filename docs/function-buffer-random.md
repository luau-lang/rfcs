# buffer.random

## Summary

Add `buffer.random` as a function to generate cryptographically secure random bytes.

## Motivation

There is no mechanism in Luau for cryptographically secure random bytes, which is a requirement for cryptographic libraries implemented in Luau.

Other scripting languages support mechanisms to achieve this, for example [JavaScript's `crypto.getRandomValues`](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues), [Python's `secret` library](https://docs.python.org/3/library/secrets.html), and [PHP's `random_bytes`](https://www.php.net/manual/en/function.random-bytes.php).

## Design

The proposal adds the following standard library function:

`buffer.random(b: buffer, offset: number, count: number): ()`

Set 'count' bytes in the buffer from specified offset to securely generated random bytes. (`/dev/urandom` on Linux, as example)

A maximum of `65536` bytes for the 'count' parameter is to be imposed to prevent accidental entropy exhaustion.

```lua
local b = buffer.create(32)
buffer.random(b, 0, 32)

-- Use as needed, for example as a cryptographic key or nonce.
```

## Drawbacks

The introduced function can expose mechanisms for draining available entropy on the system, although [resource exhaustion is considered out of scope of the Luau security model](https://luau-lang.org/sandbox#interrupts).

## Alternatives

Existing library options, like `math.random` (or the `Random` library specific to Roblox) are not sufficient for cryptographic purposes.