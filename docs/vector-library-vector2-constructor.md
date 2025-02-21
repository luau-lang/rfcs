# 2-component vector constructor

**Status**: Implemented

## Summary

Add a constructor for initializing vectors from two components.

## Motivation

Currently the vector constructor supports only initializing vectors with three components (and optionally four if 4-wide vectors are enabled). However, there are problem domains such 2D graphics and graphical user interfaces which would benefit from a two component constructor. The benefits are twofold: it would improve code ergonomics because the third component does not need to be given when it is irrelevant or implicitly zero. Secondly, it would improve the performance of vector creation because there is one less argument to pass in the 2D case.

Existing vector library functions such as `vector.dot`, `vector.angle`, `vector.normalize`, `vector.min`, `vector.max`, `vector.abs` and `vector.clamp` already work out of the box in 2D as long as the unused components are zeroes. The functions also retain zeroes allowing function calls to be chained without issues. Out of the current functions, only `vector.cross` is not meaningful in 2D.

The constructor in 4-wide mode already sets the fourth component to zero when unspecified. Therefore, the 2-component constructor working identically for the unused components can be treated as a natural extension.

Game engines that use a Z up coordinate system would also benefit from this change because the new constructor could be used for topographical positions in the world by providing X and Y only.

## Design

A new vector constructor `vector.create(x: number, y: number): vector` will be added. The third component, and fourth component with 4-wide vectors, will be zero initialized.

## Drawbacks

Accidentally omitting the third component becomes a possibility because the vector constructor would no longer raise an error when only two components are provided. However, the benefits seem to outweight this small inconvenience.

## Alternatives

An alternative would be to add a new constructor with a different name (for example, `vector.create2`).

Another considerably more complex alternative is to add a new, separate vector type for 2D vectors. This would add significant bloat to the C API and type system in general, so it is unfeasible in practice.

Finally, an option is to not add the new constructor. In this case applications working in 2D would not get the ergonomic and performance benefits discussed above.