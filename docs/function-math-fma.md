# math.fma

## Summary

Add `fma` as a function to the math library, computing the [Fused multiply–add](https://en.wikipedia.org/wiki/Multiply%E2%80%93accumulate_operation#Fused_multiply%E2%80%93add) operation, following the appropriate IEEE standard: [IEEE 754-2008](https://ieeexplore.ieee.org/document/4610935) *[(extension)](https://ieeexplore.ieee.org/document/5487502)*.

## Motivation

Fused multiply-add, also known as multiply-accumulate and often abbreviated as `fma` or `mad`, is a computation which condenses the operations `(a × b) + c`, which normally would require a `MULSD` and `ADDSD` instruction, into a single processor instruction.

This operation is commonly used in calculations such as dot products, cross products, quaternion rotations, and matrix multiplications.

The *two* advantages of this operation are as follows:
1. Floating point rounding only occurs at the end of the instruction, allowing for enhanced precision over computing two separate instructions.
2. Instruction count is reduced for math-intensive code, resulting in smaller code size and better performance.

   The example below is a dot product between two 4-dimensional vectors:

   ```cpp
   double dot(
     double ax, double ay, double az, double aw,
     double bx, double by, double bz, double bw
   )
   {
     return
         (ax * bx)
       + (ay * by)
       + (az * bz)
       + (aw * bw);
   }
   ```

   This computation results in a total of `7` math instructions
     ```asm
     MULSD xmm0, xmm4             ; ax * bx
     MULSD xmm1, xmm5             ; ay * by
     MULSD xmm2, xmm6             ; az * bz
     MULSD xmm3, xmm7             ; aw * bw
  
     ADDSD xmm0, xmm1             ; (ax * bx) + (ay * by)
     ADDSD xmm0, xmm2             ; + (az * bz)
     ADDSD xmm0, xmm3             ; + (aw * bw)
     ```
   Algorithm simplified with `fma`:
     ```cpp
     double dot(
       double ax, double ay, double az, double aw,
       double bx, double by, double bz, double bw
     )
     {
       return fma(
         ax,  bx, fma(
         ay,  by, fma(
         az,  bz,
         aw * bw
         )
         )
       );
     }
     ```
   The optimization will reduce to `4` math instructions:
     ```asm
     MULSD       xmm3, xmm7       ; aw * bw
     VFMADD231SD xmm2, xmm6, xmm3 ; az * bz + (aw * bw)
     VFMADD231SD xmm1, xmm5, xmm2 ; ay * by + (az * bz + aw * bw)
     VFMADD231SD xmm0, xmm4, xmm1 ; ax * bx + (ay * by + az * bz + aw * bw)
     ```  

There is also the potential of updating libraries such as `vector` to make use of `math.fma` internally, and other cases such as Roblox's `CFrame` could also benefit from this change.

## Design

Introduction of a new function, `math.fma`, which will perform the equivalent of the following operation:
```luau
function math.fma(a: number, b: number, c: number): number
  return (a * b) + c;
end
```

The implementation of the function (internally) should make use the [`math.fma`](https://en.cppreference.com/w/c/numeric/math/fma) operation from the `<math.h>` library (`<cmath>` in C++).

When generating `native` code with `--!native` or `@native`, the operation should optimize the use of the [FMA instruction set](https://en.wikipedia.org/wiki/FMA_instruction_set) on supported devices, but fall back to `MULSD`, `ADDSD`, and `SUBSD` instructions for unsupported devices.

## Drawbacks

- `fma` instructions are not supported on all devices, so the benefits of using `math.fma` may be limited to certain hardware and result in inconsistent performance across devices.
- Although it is possible to automatically convert instances of `(a × b) + c` and `c + (a × b)` into `fma` instructions, it is *ill-advised*.

  Automatic conversions will harm user code, such as `sqrt(x * x - y * y)`
  
  The compiler would attempt to optimize the interior of `sqrt(x * x - y * y)` into `(x * x) - (y * y)` and eventually `fma(x, x, -(y * y))`, which should compile to the following arithmetic instructions:
    ```asm
    MULSD       xmm1, xmm1       ; y * y
    VFMSUB231SD xmm0, xmm0, xmm1 ; x * x - (y * y)
    ```

  This is problematic, as there is a gap between the precision of the operations. `a * b` might produce a rounding error which is different from `a * b - c`, and a negative value could unintentionally be introduced to the square root, even if `x == y`, resulting in an error which previously would not have existed.

  The *original* interpretation would be the following:
    ```asm
    MULSD xmm0, xmm0             ; x * x
    MULSD xmm1, xmm1             ; y * y

    SUBSD xmm0, xmm1             ; (x * x) - (y * y)
    ```

  There is no complication in this scenario, as both operations are identical and would result in the same rounding error.

  Overall, it is better to allow the developer to manually decide which operations they choose to optimize, as the compiler will not understand the context, importance, or order of the operations and the rationale behind their arrangements.

  As a consequence, `math.fma` must manually be invoked, and existing code would not benefit from any performance improvements.

- The use of `math.fma` may result in confusing code if it is not cleanly implemented by the developer.
- Additional function added to the `math` library, which may only see use from more experienced developers that understand the micro-optimization of mathematical operations.
- Documentation for `math.fma`, particularly its benefits, pitfalls, and behavior are required to well explain when, why, and how to use `math.fma`.

## Alternatives

- Not implementing `math.fma`, remaining with the lesser optimized method of manually computing `(a * b) + c`
- Implementing sister functions, such as `math.fms` and/or `math.fma231`, to make full use of the [FMA instruction set](https://en.wikipedia.org/wiki/FMA_instruction_set), allowing for more delicate optimization of operations, and finer control over the order and structure of operations.
