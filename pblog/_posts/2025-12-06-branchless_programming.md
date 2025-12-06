---
layout: pblog_post
title: Branchless Programming Abstractions
description: A short writeup on language abstractions for facilitating branchless programming.
---

One of the simplest and most well-known examples of a hardware detail that's
relevant to performance is branch misprediction. The fundamental issue is that
mispredicted branches lead to wasted work and pipeline stalls.

If we're thinking of what a language might do if it was designed with branch
prediction in mind, we'd primarily be looking for ways that it could either help
us make unpredictable branches more predictable, or help us avoid needing to use
a branch in the first place.

## Branch Hints
One way to make branches more predictable is to inform our CPUs of what
direction we expect branches to go. Some ISAs such as x86 and ARM offer
mechanisms for static branch hints, essentially meaning that the instruction
stream informs the CPU that it should predict a branch will go a particular way.
It's worth noting that these hints are ignored on almost all mainstream x86
hardware, so the utility of this feature is limited. That said, on Intel's
Redwood Cove machines, hints that a branch is taken are respected, but hints
that a branch is not taken are still ignored. Regardless, this suggests that
perhaps we should still try creating abstractions over this ISA feature, should
it ever happen to be useful.

If we take a look at C++ 20, we see a language that has added a feature that
could potentially be used to leverage this. The attributes `[[likely]]` and
`[[unlikely]]` can be applied to branches as hints to the compiler, which may
then pass those hints onto the machine code. Although there is not guarantee
that this will be done, these attributes also serve another purpose. The
compiler can use these to decide how to order the branches' code. When a modern
Intel CPU encounters a branch for which it has no local history to base a branch
prediction off of, it will instead base its prediction on whether the jump is
forward or backwards. Backwards jumps are assumed to be taken (such as in the
case of loops), while jumps forward are assumed to not be taken. Therefore, by
swapping the way that the branches' code are ordered within the instruction
stream can be used as a mechanism for providing the branch predictor with a
hint.

While I think this is a step in the right direction, I don't think that it's
quite enough. After all, if you can reliably predict what direction a branch
will go statically, it's not an unpredictable branch. This might save you the
occasional misprediction, but it won't do anything about the branches which are
truly problematic, the ones which are completely unpredictable.

In such cases, I think an `[[unpredictable]]` attribute would be appropriate.
This could act as a hint to the compiler that it should be more aggressive about
finding ways to eliminate the associated branch (at least, when targeting
machines with branch prediction. On a machine that does not, it seems reasonable
for the implementation to be permitted to ignore this). It may be useful if the
compiler could optionally emit a warning in cases where it's unable to
accomplish this, and some diagnostics which indicate what the particular
roadblock(s) were.

Taking this one step further, you might even consider a `[[branchless]]`
attribute which requires that the compiler avoid emitting a branch. If the
compiler is unable to do so, then an error would be raised. This does raise the
question of what kind of branches would be mandatory for an implementation to
handle as there would be a need for different compilers to handle a common set
of branches. Otherwise, code that uses this attribute would not be portable. In
the general case, this is quite a difficult challenge. If calling a function
whose definition is not available when compiling the current translation unit,
then said function may have side effects which cannot be accounted for. Hence,
some set of pure operations, or operations whose semantics the compiler is fully
aware of would need to be established as being a minimal set that could be used
within an if-statement tagged with `[[branchless]]`.

All that said, what would be really nice would be for the language to enable us
to eliminate branches for ourselves.

## Branchless Programming
Branchless programming encompasses a broad range of techniques which are applied
to remove conditional jumps from our code, while maintaining its functionality.
If a language offered facilities for branchless programming, it could become a
way for programmers to explicitly avoid code which contains branches.

### Conditional Instructions
It's not uncommon for modern ISAs to help us write branchless code by offering
conditional instructions.

On x86 we have an instruction called `cmov`, which conditionally performs a move
to a register based on the state of the CPU's flags. This instruction can be
used to replace simple branches, such as if-statements that select between
multiple values. Since the instruction is always executed, it does not cause the
scheduler to back up. Not only that, but with Intel's upcoming APX extension,
x86 will be getting other conditional operations: loads, stores, integer
comparisons and bitwise tests.

These kinds of instructions are not unique to x86. ARM has a conditional
execution feature where a wide range of instructions may have their execution
predicated on the state of CPU flags. MIPS has had FPU conditional moves via
`movt`, `movf`, `movz` and `movn` since MIPS IV. PowerISA has `isel`. RISC-V has
`czero.eqz` and `czero.nez` instructions that conditionally zero out a register.
So then, I think it's only fitting that these features be abstracted by a modern
programming language.

### Blending & Friends
A simple way to abstract these features would be for programming languages to
offer a simple function: `blend(bool b, T true_val, T false_val)` where `T` is a
primitive type. This implements an operation which selects between two values
predicated on a Boolean. The idea here is that on machine architectures where
branch misprediction is a relevant issue, the compiler would be required to
avoid implementing the function via a branch. Instead, it would have to rely on
instructions like those previously mentioned.

Indeed, this operation is actually found on modern ISAs as well. x86 SIMD has
many SIMD instructions which implement a blend operation on a per lane basis,
the earliest of these being introduced with SSE4.1. More recently, AVX-512
features a blend masking feature, which essentially allows almost any AVX-512
instruction to also perform a blend operation between the instruction's computed
result, and some other value from the corresponding lane within another vector
operand. AVX-512 also introduced the `vpternlog*` instructions which can be used
to perform a blend operation at a bitwise granularity. Over on ARM, Neon
features the `bsl` instruction, which performs a bitwise blend, and SVE features
`sel`, which performs a per-lane blend. SVE has SIMD merge predication, which is
similar to AVX-512's blend masking and can therefore also be abstracted by this
function.

In the same vein, I would like to see two other more functions, which I'll call
`keep(bool b, T value)` and `clear(bool b, T value)`. `keep` would return the
value passed in if the Boolean is true and return zero otherwise. `clear` would
return zero if the Boolean is true and return the value passed in otherwise.
These can be though of as special cases of the aforementioned `blend` operation
where one of the values being blended is known to be zero. These special cases
are fairly common (in my subjective experience), may be implemented just as or
more simply than a blend (depending on the target ISA), and can sometimes can do
a better job of reflecting ISA features.

For example, the aforementioned `czero.eqz` and `czero.nez` instructions on
RISC-V have the semantics of these operation. On x86 with SSE and AVX SIMD,
these operations can be implemented using just bitwise AND, and ANDNOT
operations since SSE and AVX comparisons return lanes with either all bits set,
or all bits cleared. On ARM, a bitwise AND or a bitwise clear via `bic` can be
used to the same effect. Indeed, `bic` is short for bitwise clear, hinting at
the instruction's utility for this purpose. With x86's AVX-512, its zero masking
can be abstracted by a `keep` operation. It's the same story for ARM SVE's
zeroing predication.

In short, these three operations can be easily implemented in a branchless
manner across a wide range of ISAs in both scalar and vector code because they
reflect features that all these ISAs offer in both the scalar and vector
domains. I think that this makes them useful improvements for the purposes of
enabling programmers to write fast code, to leverage ISA features, and to write
code in a way that fairly directly abstracts over those ISA features.

When thinking about the requirement that these functions be implemented without
a conditional jump, I considered that possibility that this requirement could be
relaxed when targeting simpler machines that do not feature pipelined execution
and which therefore don't suffer from branch misprediction. After all, if at
least one of the values being blended is somewhat expensive to compute, it seems
a reasonable optimization to only compute values which will actually be used.
Despite this, I think it would still be beneficial to maintain this requirement
even on these machines so that it could reliably be used for implementing
timing-safe functions. That is, functions whose execution time is not dependent
on their inputs and whose execution time an therefore not be targeted by a
timing side-channel attack. In the case that the programmer doesn't care about
whether this is implemented in a branchless fashion, they could always use a
ternary operator anyways.

### Bitwise Blend & Friends
You may have noticed that I mixed together operations that operate at bitwise
granularity with those that operate at a lane-wise granularity. I mentioned them
together because I was presenting lane-wise abstractions, and the bitwise blends
can naturally be used to implement lane-wise blends. However, bitwise blends may
be useful in their own right so I would suggest bitwise variants of these
instructions such as:

```
T blend_bits(T mask, T true_value, T false_false)
T keep_bits(T mask, T true_value, T false_false)
T clear_bits(T mask, T true_value, T false_false)
```

Where `T` is an integral type.

### Branchless Tricks
It's worth considering the fact that there are a number of tricks for performing
specific operations in a branchless fashion. However, as I write this, I don't
see a need for programming languages to specifically abstract over these. In
particular, if the programmer expressed these conditional operations using the
aforementioned primitives, a compiler could reasonably apply peephole
optimizations to utilize these tricks where applicable.

For example, suppose that someone wrote on of the following:

```
a = blend(condition, a, a + b)
a += keep(condition, b)
```

If this code was targeting x86 and the compiler was vectorizing it with SSE, it
could implement either by computing the bitwise AND of the add operation's
second operand and the condition before performing the actual addition.(Recall
that conditions are indicated by a lane which has either all bits cleared to
indicate false, or all bits set to indicate true.)

```
a = blend(condition, a, a + b)
a += condition & b
```

So long as a list of these branchless tricks could be collected and get
corresponding optimizations implemented by mainstream compilers, I think these
approach could pan out.

## A Note On Implications For SIMD
Although the primary utility of these operations has been presented here as
avoiding the need to emit branches, there is another major benefit that's
illustrated by the example provided in the previous section. It's that code that
is branchless is also easier to vectorize. By replacing branches with these
operations, we are also replacing an algorithmic primitive that doesn't exist
in the SIMD domain with one that does.

## Transforming Branches
Something that's worth mentioning is that if-else statements that are
well-behaved enough can be transformed to be expressed in terms of blend
operations. You may want to check out [the script](/talks/x86_simd_script.html)
and [the
slides](https://drive.google.com/file/d/1aTn1_MqXcU-Ih6Y10PJ8tnzChbBnMb0z/view)
for a talk I gave a while back for a quick example of what that might look like.
In particular jump to the section titled "If Statements".
