# Assembly Optimization Guide for Graviton Arm64 Processors

## Introduction

This document is a reference for software developers who want to optimize code
for Graviton Arm64 processors at the assembly level. Some of the documented
transforms could be applied to optimize code written in higher level languages
like C or C++, and in that respect this document is not restrictive to assembly
level programming. This guide isn’t intended cover every aspect of writing in
assembly, such as assembler syntax or ABI details.

The code patterns in this document could also be useful to spot inefficient code
generated for the hot code. Profilers such as Linux perf allow the inspection of
the assembly level code. During the review of the hot code, software developers
can reference this guide to find better ways to optimize inner loops.

Some techniques for writing optimized assembly:
1. [Be aware of instruction level parallelism](#instruction-level-parallelism)
1. [Split Data Dependency Chains](#split-data-dependency-chains)
1. [Modulo Scheduling](#modulo-scheduling)


We will be adding more sections to this document soon, so check back!

## Instruction Level Parallelism
When writing in C, or any higher level language, the programmer writes a sequence
of statements which are presumed to execute in strict sequence. In the following
example, each statement logically occurs after the previous one.

```c
int foo (int a, int b, int c)
{
    if (c < 0) {
        int d = a + b;
        return d;
    } else {
        int e = a * b;
        return e;
    }
}
```

However when we compile this code with gcc version 7.3 with `-O1`, this is the output:
```
foo(int, int, int):
    add     w3, w0, w1
    mul     w0, w0, w1
    cmp     w2, 0
    csel    w0, w0, w3, ge
    ret
```

Here we can see the add and the multiply instruction are executed independently
of the result of `c < 0`. The compiler knows that the CPU can execute these
instructions in parallel. If we use the machine code analyzer from the LLVM
project, we can can see what is happening. We will use Graviton2 as our target
platform, which uses the Neoverse-N1 core.

`llvm-mca -mcpu=neoverse-n1 -timeline -iterations 1 foo.s`
```
...
Timeline view:
Index     012345

[0,0]     DeER .   add  w3, w0, w1
[0,1]     DeeER.   mul  w0, w0, w1
[0,2]     DeE-R.   cmp  w2, #0
[0,3]     D==eER   csel w0, w0, w3, ge
[0,4]     DeE--R   ret
...
```

In this output we can see that all five instructions from this function are decoded
in parallel in the first cycle. Then in the second cycle the instructions for which
all dependencies are already resolved begin executing. There is only one instruction
which has dependencies: the conditional select, `csel`. The four remaining
instructions all begin executing, including the `ret` instruction at the end. In
effect, nearly the entire C function is executing in parallel. The
`csel` has to wait for the result of the comparison instruction to check
`c < 0` and the result of the multiplication. Even the return has
completed by this point and the CPU front end has already decoded
instructions in the return path and some of them will already be
executing as well.

It is important to understand this when writing assembly. Instructions
do not execute in the order they appear. Their effects will be logically
sequential, but execution order will not be.

When writing assembly for arm64 or any platform, be aware that the CPU has more
than one pipeline of different types. The types and quantities are outlined in
the Software Optimization Guide, often abbreviated as SWOG. The guide
for each Graviton processor is linked from the [main page of this
technical guide](README.md). Using this knowledge, the programmer can arrange instructions of
different types next to each other to take advantage of instruction level
parallelism, ILP. For example, interleaving load instructions with vector or
floating point instructions can keep both pipelines busy.


## Split Data Dependency Chains

As we saw in the last section, Graviton has multiple pipelines or execution units which can execute instructions. The instructions may execute in parallel if all of the input dependencies have been met but a series of instructions each of which depend on a result from the previous one will not be able to efficiently utilize the resources of the CPU.

For example, a simple C function which takes 64 signed 8-bit integers and adds them all up into one value could be implemented like this:

```
int16_t add_64(int8_t *d) {
    int16_t sum = 0;
    for (int i = 0; i < 64; i++) {
        sum += d[i];
    }
    return sum;
}
```

We could write this in assembly using NEON SIMD instructions like this:

```
add_64_neon_01:
    ld1     {v0.16b, v1.16b, v2.16b, v3.16b} [x0]   // load all 64 bytes into vector registers v0-v3
    movi    v4.2d, #0                               // zero v4 for use as an accumulator
    saddw   v4.8h, v4.8h, v0.8b                     // add bytes 0-7 of v0 to the accumulator
    saddw2  v4.8h, v4.8h, v0.16b                    // add bytes 8-15 of v0 to the accumulator
    saddw   v4.8h, v4.8h, v1.8b                     // ...
    saddw2  v4.8h, v4.8h, v1.16b
    saddw   v4.8h, v4.8h, v2.8b
    saddw2  v4.8h, v4.8h, v2.16b
    saddw   v4.8h, v4.8h, v3.8b
    saddw2  v4.8h, v4.8h, v3.16b
    addv    h0, v4.8h                               // horizontal add all values in the accumulator to h0
    fmov    w0, h0                                  // copy vector registor h0 to general purpose register w0
    ret                                             // return with the result in w0
```

In this example, we use a signed add-wide instruction which adds the top and bottom of each register to v4 which is used to accumulate the sum. This is the worst case for data dependency chains because every instruction depends on the result of the previous one which will make it impossible for the CPU to achieve any instruction level parallelism. If we use `llvm-mca` to evaluate it we can see this clearly.

```
Timeline view:
                    0123456789
Index     0123456789          0123456

[0,0]     DeeER.    .    .    .    ..   movi    v4.2d, #0000000000000000
[0,1]     D==eeER   .    .    .    ..   saddw   v4.8h, v4.8h, v0.8b
[0,2]     D====eeER .    .    .    ..   saddw2  v4.8h, v4.8h, v0.16b
[0,3]     D======eeER    .    .    ..   saddw   v4.8h, v4.8h, v1.8b
[0,4]     D========eeER  .    .    ..   saddw2  v4.8h, v4.8h, v1.16b
[0,5]     D==========eeER.    .    ..   saddw   v4.8h, v4.8h, v2.8b
[0,6]     D============eeER   .    ..   saddw2  v4.8h, v4.8h, v2.16b
[0,7]     D==============eeER .    ..   saddw   v4.8h, v4.8h, v3.8b
[0,8]     D================eeER    ..   saddw2  v4.8h, v4.8h, v3.16b
[0,9]     D==================eeeeER..   addv    h0, v4.8h
[0,10]    D======================eeER   fmov    w0, h0
[0,11]    DeE-----------------------R   ret
```


One way to break data dependency chains is to use commutative property of addition and change the order which the adds are completed. Consider the following alternative implementation which makes use of pairwise add instructions.

```
add_64_neon_02:
    ld1     {v0.16b, v1.16b, v2.16b, v3.16b} [x0]

    saddlp  v0.8h, v0.16b                   // signed add pairwise long
    saddlp  v1.8h, v1.16b
    saddlp  v2.8h, v2.16b
    saddlp  v3.8h, v3.16b                   // after this instruction we have 32 16-bit values

    addp    v0.8h, v0.8h, v1.8h             // add pairwise again v0 and v1
    addp    v2.8h, v2.8h, v3.8h             // now we are down to 16 16-bit values

    addp    v0.8h, v0.8h, v2.8h             // 8 16-bit values
    addv    h0, v4.8h                       // add 8 remaining values across vector
    fmov    w0, h0

    ret
```

In this example the first 4 instructions after the load can execute independently and then the next 2 are also independent of each other.  However, the last 3 instructions do have data dependencies on each other. If we take a look with `llvm-mca` again, we can see that this implementation takes 17 cycles (excluding the initial load instruction common to both implementations) and the original takes 27 cycles.

```
Timeline view:
                    0123456
Index     0123456789

[0,0]     DeeER.    .    ..   saddlp    v0.8h, v0.16b
[0,1]     DeeER.    .    ..   saddlp    v1.8h, v1.16b
[0,2]     D=eeER    .    ..   saddlp    v2.8h, v2.16b
[0,3]     D=eeER    .    ..   saddlp    v3.8h, v3.16b
[0,4]     D==eeER   .    ..   addp      v0.8h, v0.8h, v1.8h
[0,5]     D===eeER  .    ..   addp      v2.8h, v2.8h, v3.8h
[0,6]     D=====eeER.    ..   addp      v0.8h, v0.8h, v2.8h
[0,7]     .D======eeeeeER..   addv      h0, v0.8h
[0,8]     .D===========eeER   fmov      w0, h0
[0,9]     .DeE------------R   ret
```


## Modulo Scheduling

When writing a loop in assembly that contains several dependent steps in each
iteration but there are no dependencies between iterations (or only superficial
dependencies), modulo scheduling can improve performance. Modulo scheduling is
the processes of combining loop unrolling with the interleaving of steps from
other iterations of the loop. For example, suppose we have a loop which has four
steps, A, B, C, and D. We can unroll that loop to execute 4 iterations in
parallel with vector instructions such that we complete steps like this: A₀₋₃,
B₀₋₃, C₀₋₃, D₀₋₃. However this construction still has a dependency chain which
the last section taught us to avoid. We can break that chain by scheduling the
steps with modulo scheduling.

```
A1
B1 A2
C1 B2 A3
D1 C2 B3 A4
   D2 C3 B4
      D3 C4
         D4
```

Take the following example written in C. I noted step A through step D in the
source, but the lines in C don’t map cleanly onto the steps I selected for
implementation in assembly.

```
void modulo_scheduling_example_c_01(int16_t *destination, int dest_len,
         int8_t *src, uint32_t *src_positions,
         int16_t *multiplier)
{
    int i;
    for (i = 0; i < dest_len * 8; i++)
    {
        // step A: load src_position[i] and multiplier[i]
        int src_pos = src_positions[i];
        // step B: load src[src_pos] and copy the value to a vector register
        // step C: sign extend int8_t to int16_t, multiply, and saturate-shift right
        int32_t val = multiplier[i] * src[src_pos];
        destination[i] = min(val >> 3, (1 << 15) - 1);
        // step D: store the result to destination[i]
    }
}
```

A first pass implementation of this function using Neon instructions to unroll
the loop could look like this:

<details>
<summary>
 Click here to see the (long) assembly implementation.
</summary>

```
    // x0: destination
    // w1: dest_len
    // x2: src
    // x3: src_positions
    // x4: multiplier
modulo_scheduling_example_asm_01:
    stp x19, x20, [sp, #-16]!

1:
    // src_pos = src_positions[i]
    ldp w5, w6, [x3]
    ldp w7, w8, [x3, 8]
    ldp w9, w10, [x3, 16]
    ldp w11, w12, [x3, 24]
    add x3, x3, #32

    // src[src_pos]
    ldrb w13, [x2,  w5, UXTW]
    ldrb w14, [x2,  w6, UXTW]
    ldrb w15, [x2,  w7, UXTW]
    ldrb w16, [x2,  w8, UXTW]

    ldrb w17, [x2,  w9, UXTW]
    ldrb w18, [x2, w10, UXTW]
    ldrb w19, [x2, w11, UXTW]
    ldrb w20, [x2, w12, UXTW]

    // copy to vector reg
    ins v0.8b[0], w13
    ins v0.8b[1], w14
    ins v0.8b[2], w15
    ins v0.8b[3], w16

    ins v0.8b[4], w17
    ins v0.8b[5], w18
    ins v0.8b[6], w19
    ins v0.8b[7], w20

    // sign extend long, convert int8_t to int16_t
    sxtl v0.8h, v0.8b

    // multiplier[i]
    ld1 {v1.8h}, [x4], #16

    // multiply
    smull  v2.4s, v1.4h, v0.4h
    smull2 v3.4s, v1.8h, v0.8h

    // saturating shift right
    sqshrn  v2.4h, v2.4s, #3
    sqshrn2 v2.8h, v3.4s, #3

    st1 {v2.8h}, [x0], #16

    subs w1, w1, #1
    b.gt 1b

    ldp x19, x20, [sp], #16

    ret
```
</details>

Lets’s benchmark both implementations and see what the results are:

```
modulo_scheduling_example_c_01 - runtime: 1.598776 s
modulo_scheduling_example_asm_01 - runtime: 1.492209 s
```

The assembly implementation is faster, which is great, but lets see how we can
do better. We could take this a step further and employ modulo scheduling. This
increases the complexity of the implementation, but for hot functions which
execute many iterations, this may be worth it. In order to make this work,
we have to prime the loop with a prologue section:

```
A1
B1 A2
C1 B2 A3
```

Then we run a steady state loop:

```
D1 C2 B3 A4
```

And finally an epilogue to complete the iterations:

```
   D2 C3 B4
      D3 C4
         D4
```

In order to make this work, each step must consume all of the outputs of the
previous step and not use as temporary registers any of the inputs or outputs of
other steps. For this example, step A loads the value of `multiplier[i]` but it
is not consumed until step C, so in step B we simply copy its value with a `mov`
instruction. This implementation could look like this:

<details>
<summary>
See the implementations of each step whicih are the same as above,
(hidden for brevity).
</summary>

```
.macro modulo_scheduling_example_asm_step_A
    // src_pos = src_positions[i]
    ldp w5, w6, [x3]
    ldp w7, w8, [x3, 8]
    ldp w9, w10, [x3, 16]
    ldp w11, w12, [x3, 24]
    add x3, x3, #32

    // multiplier[i]
    ld1 {v5.8h}, [x4], #16
.endm

.macro modulo_scheduling_example_asm_step_B
    mov v1.16b, v5.16b

    // src[src_pos]
    ldrb w13, [x2,  w5, UXTW]
    ldrb w14, [x2,  w6, UXTW]
    ldrb w15, [x2,  w7, UXTW]
    ldrb w16, [x2,  w8, UXTW]

    ldrb w17, [x2,  w9, UXTW]
    ldrb w18, [x2, w10, UXTW]
    ldrb w19, [x2, w11, UXTW]
    ldrb w20, [x2, w12, UXTW]

    // copy to vector reg
    ins v0.8b[0], w13
    ins v0.8b[1], w14
    ins v0.8b[2], w15
    ins v0.8b[3], w16

    ins v0.8b[4], w17
    ins v0.8b[5], w18
    ins v0.8b[6], w19
    ins v0.8b[7], w20
.endm

.macro modulo_scheduling_example_asm_step_C
    // sign extend long, convert int8_t to int16_t
    sxtl v0.8h, v0.8b

    // multiply
    smull  v2.4s, v1.4h, v0.4h
    smull2 v3.4s, v1.8h, v0.8h

    // saturating shift right
    // val = min(val >> 3, (1 << 15) - 1)
    sqshrn  v4.4h, v2.4s, #3
    sqshrn2 v4.8h, v3.4s, #3
.endm

.macro modulo_scheduling_example_asm_step_D
    // step D
    // destination[i] = val
    st1 {v4.8h}, [x0], #16
.endm
```

</details>

```
.global modulo_scheduling_example_asm_02
modulo_scheduling_example_asm_02:
    cmp w1, #4
    b.lt modulo_scheduling_example_asm_01

    stp x19, x20, [sp, #-16]!

    modulo_scheduling_example_asm_step_A // A1
    modulo_scheduling_example_asm_step_B // B1
    modulo_scheduling_example_asm_step_A // A2
    modulo_scheduling_example_asm_step_C // C1
    modulo_scheduling_example_asm_step_B // B2
    modulo_scheduling_example_asm_step_A // A3

1:  //// loop ////
    modulo_scheduling_example_asm_step_D
    modulo_scheduling_example_asm_step_C
    modulo_scheduling_example_asm_step_B
    modulo_scheduling_example_asm_step_A

    sub w1, w1, #1
    cmp w1, #3
    b.gt 1b
    //// end loop ////

    modulo_scheduling_example_asm_step_D // DN-2
    modulo_scheduling_example_asm_step_C // CN-1
    modulo_scheduling_example_asm_step_B // BN
    modulo_scheduling_example_asm_step_D // DN-1
    modulo_scheduling_example_asm_step_C // CN
    modulo_scheduling_example_asm_step_D // DN

    // restore stack
    ldp x19, x20, [sp], #16
    ret
```

If we benchmark these functions on Graviton3, this is the result:

```
modulo_scheduling_example_c_01 - runtime: 1.598776 s
modulo_scheduling_example_asm_01 - runtime: 1.492209 s
modulo_scheduling_example_asm_02 - runtime: 1.583097 s
```

It’s still faster than the C implementation, but it’s slower than our first
pass. Let’s add another trick. If we check the software optimization guide for
Graviton2, the Neoverse N1, we see that the `ins` instruction has a whopping
5-cycle latency with only 1 instruction per cycle of total throughput. There are
8 `ins` instructions which will take at least 13 cycles to complete, the whole
time tying up vector pipeline resources.

If we instead outsource that work to the load and store pipelines, we can leave
the vector pipelines to process the math instructions in step C. To do this,
after each byte is loaded, we write them back to memory on the stack with `strb`
instructions to lay down the bytes in the order that we need them to be in the
vector register. Then in the next step we use a ld1 instruction to put those
bytes into a vector register. This approach probably has more latency (but I
haven’t calculated it), but since the next step which consumes the outputs is
three steps away, it leaves time for those instructions to complete before we
need those results. For this technique, we revise only the macros for steps B
and C:


```
.macro modulo_scheduling_example_asm_03_step_B
    mov v4.16b, v5.16b

    strb w13, [sp, #16]
    strb w14, [sp, #17]
    strb w15, [sp, #18]
    strb w16, [sp, #19]

    strb w17, [sp, #20]
    strb w18, [sp, #21]
    strb w19, [sp, #22]
    strb w20, [sp, #23]
.endm

.macro modulo_scheduling_example_asm_03_step_C
    mov v1.16b, v4.16b
    ldr d0, [sp, #16]
.endm
```

This technique gets the best of both implementations and running the benchmark
again produces these results:

```
modulo_scheduling_example_c_01 - runtime: 1.598776 s
modulo_scheduling_example_asm_01 - runtime: 1.492209 s
modulo_scheduling_example_asm_02 - runtime: 1.583097 s
modulo_scheduling_example_asm_03 - runtime: 1.222440 s
```

Nice. Now we have the fastest implementation yet with a 24% speed improvement.
But wait! What if the improvement is only from the trick where we save the
intermediate values on the stack in order to get them into the right order and
modulo scheduling isn’t helping. Let’s benchmark the case where we use this
trick without modulo scheduling.

```
modulo_scheduling_example_asm_04 - runtime: 2.106044 s
```

Ooof. That’s the slowest implementation yet. The reason this trick works is
because it divides work up between vector and memory pipelines and stretches out
the time until the results are required to continue the loop. Only when we do
both of these things, do we achieve this performance.

I didn’t mention SVE gather load instructions on purpose. They would most
definitely help here, and we will explore that in the future.
