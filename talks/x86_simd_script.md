---
layout: singular_page
title: The Neglected Majority of x86 - SIMD
---

## x87 and MMX History

When x86 was originally created, it lacked dedicated floating-point facilities.
To perform floating-point arithmetic, you could either leverage a software
implementation, or you could use a math co-processor which added hardware
support for floating-point instructions. Most notable of these, were those from
the x87 line, which came with their own correspondingly named extension to the
instruction set. x87 operated on an 80-bit floats with 15-bit exponents and
64-bit significands. Since x86's general-purpose registers are only 64-bits
wide, x87 had to introduce new registers, 80 bits wide to hold these 80-bit
floats.

In 1997, these registers were given a new purpose when x86 got an extension
known as MMX, or multimedia extension. MMX added instructions that would treat
the x87 registers as being 64-bits wide, and as being packed with integers.
Essentially, the registers would hold an array of eight 8-bit integers, four
16-bit integers, or two 32-bit integers. After populating these registers in
such a fashion, MMX instructions could be applied to perform pair-wise
additions, subtractions, multiplications, shifts, comparisons, and more. With
MMX, you could now process multiple data elements in parallel within a single
CPU instruction. This was x86's first time getting SIMD capabilities.

## Defining SIMD & Other Vocab

The term SIMD is an acronym which stands for single-instruction, multiple-data.
It comes from a system for categorizing computer architectures known as Flynn's
taxonomy. This system places machines into one of four categories based on
whether they process singular or multiple instruction/data streams.

Under this original definition of the term, SIMD as a concept can refer to a
broad range of architectural techniques for parallelism. But in practical terms
it tends refers to the techniques that are seen on contemporary mainstream CPUs,
based on packing data into large specialized registers whose contents are then
processed in parallel.

Adapting some math terms, it’s common to refer to the facilities that process
one data element at a time using as being scalar, and to the facilities that
process multiple elements as vector. So we might refer to SIMD registers and
SIMD instructions as vector registers and vector instructions in a synonymous
capacity. We might also say that we are vectorizing our code if we are rewriting
a function to leverage SIMD instructions.

To refer to the specific indices within a vector, we typically use the term
“lane”. So if we had a 64-bit register that contained four 16-bit integers, we
might say “The vector has four 16-bit lanes”. In a similar fashion, suppose you
had an image with four 16-bit color channels, and you had one of its pixels
loaded into the 64-bit register. We would say that, “Lane 0 contains the red
component, Lane 1 contains the green component, etc. etc.”.

There is also a need to talk about just how large these vectors can get, which
is of course not only determined by the size of the elements, but the size of
the SIMD registers, and the operations which manipulate them. For this purpose,
we talk about the “width” of a SIMD implementation, a number conventionally
measured in bits.

Typically SIMD instructions operate in a fashion such that data generally flows
within lanes. So for example, if you’re performing a typical SIMD add
instruction, there is no interaction between different lanes, only between the
same lanes within the different registers. We refer to this flow of data as
vertical.

There are however various SIMD instructions where the data flow is primarily
across lanes. We tend to refer to such operations as being horizontal. As an
example, a horizontal add operation might perform additions between adjacent
pairs of lanes within on vector, instead of between corresponding lanes in two
different vector registers.

## The Modern Landscape of SIMD

MMX may have been the beginning of SIMD on x86 but today it forms a tiny
fraction of the instruction set. x86 has continued to be extended with more and
more SIMD extensions since the late 90s. MMX is now considered outdated and its
use is discouraged, the same going for x87.

The modern landscape of x86 SIMD consists of a variety of SIMD extensions that
can be roughly grouped into a small number of families.

SSE, AVX, and AVX-512 are the families which you will find in consumer grade
hardware today. These are the ones that most programmers would consider relevant
because they’re the ones which can be reasonably leveraged for practical gain.

There is also the AMX family. I wouldn’t say it’s particularly important, but I
will address it briefly. To keep it short, AMX can be though of as SIMD that
operates on two-dimensional tiles to accelerate operations such as matrix
multiplications on 8-bit integers, half-precision floats, and brain floats.
These features are primarily designed to improve the performance of machine
learning workflows, where matrix multiplication is naturally of great
importance. It’s currently only available on Intel server machines and it seems
that it will remain that way for the foreseeable future. This narrow range of
availability coupled with the feature set being remarkably limited means AMX is
probably not something you’ll find useful.

AVX-10 is the future direction that x86 SIMD is headed today. For the time being
it’s largely a curiosity, but if you’re working on something that might benefit
from a healthy amount of planning, it might be worth keeping in the back of your
mind.

### SSE Family

Following MMX, x86 had its SIMD capabilities expanded by an entirely new
extension. This was SSE, or streaming SIMD extensions, and it introduced 128-bit
wide SIMD. Unlike MMX, SSE did not repurpose existing registers, but instead
introduced an entirely new set of eight 128-bit registers. Alongside these new
registers there were of course new instructions which operated on them.

Perhaps most notably, SSE and SSE2 gave x86 is current facilities for
manipulating single and double-precision floats respectively. Although these are
SIMD extensions, they brought both scalar and vector forms of floating-point
arithmetic. It should be noted that scalar floating-point instructions also make
use of SIMD registers. The inputs and outputs of scalar floating-point
operations are stored within the first lane of an 128-bit SIMD register.

If you compile a program for x86 using a modern compiler, it will by default use
the SSE and SSE2 facilities instead of the older x87 ones. Indeed, both SSE and
SSE2 are part of the base x86-64 instruction set so you can be confident that
you have SIMD available to you any time you’re targeting 64-bit x86 machines.

But floating-point facilities are hardly all that SSE and SSE2 brought us. The
SSE family brings a reasonable, although fragmented and incomplete set of
facilities for processing integers, with a focus on operations that are likely
to be used in applications such as image manipulation, video encoding, and audio
signal processing.

SSE3 through SSE4.1 rounded out x86’s 128-bit facilities in various ways, but
SSE4.2 is a bit of an odd ball. It primarily aims to accelerate string
processing through specialized instructions, but their performance is not great,
making them a questionable choice. But SSE4.2 does also add some instructions
for performing cyclic-redundancy checks, which do perform well and are generally
worthwhile in the right context.

Today, the entire SSE family is widely available on consumer grade hardware and
outright assuming it's presence is not unreasonable. Indeed, Windows 11 requires
SSE 4.2 be present, which also  implies the other extensions. Hence, software
builds targeting Windows 11 can confidently assume that the entire SSE family is
available. 

### AVX Family

The successor to the SSE family is the advanced vector extensions or AVX family,
which introudces 256-bit wide SIMD. While MMX may have repurposed existing
registers, and SSE introduced new registers, AVX took a hybrid approach by
instead expanding existing registers. The old xmm registers from SSE were
essentially redefined to be the lower halves of 256-bit ymm registers. In
addition to this, the amount of logical SIMD registers was doubled to a total of
16. AVX also brings a new instruction prefix called VEX for encoding SIMD
instructions. It’s applicable to 128-bit just as well as 256-bit SIMD, so they
too can benefit from the increased number of registers, and the new 3 operand
forms.

AVX can primarily be described as widening the single and double-precision float
facilities we got from the SSE family and making them available at a width of
256. AVX2 does much the same for integer operations. These extensions do
relatively little in terms of bringing completely new operations, but there is
the notable exception of 32 and 64-bit gather loads, which are basically the
SIMD versions of an indexed load. A vector full of offsets are applied to a
single pointer to enable loading from non-contiguous memory locations.

FP16C simply adds SIMD conversions between half-precision and single-precision
floats, in both directions.

FMA brings its namesake, fused multiply-add instructions. These are
floating-point operations which multiply two numbers and then add a third value
to the produce the final result. They do this without the intermediate rounding
step that would be necessary if performing two distinct operations, and they
take no more time than the multiplication does by itself.

Today, support for these four extensions is fairly widespread in consumer grade
hardware. Outright assuming that your end users will have these extensions might
be a bit excessive, but it is reasonable to expect that these would be the most
advanced SIMD capabilities that a significant portion of your end users are
likely to have.

Although nearly fifteen years old, the AVX family has continued to be extended
until recent years, and support for these more recent extensions is naturally
less common than the earlier members of this family. When it comes to these
newer extensions, they are very much still gaining ground. Some of these
extensions such as AVX-VNNI and AVX-IFMA, are essentially backports of features
introduced by AVX-512, the idea being that a machine could leverage these
features even if they only implemented 256-bit wide SIMD.

The VNNI extensions are meant to accelerate dot products on 8 and 16 bit
integers, again for the purpose of making machine learning applications runs
faster.

Some more recent extensions are named after hashing/cipher algorithms, and
straightforwardly exist to accelerate said algorithm.

### AVX-512 Family

As the name suggests, AVX-512 brings 512-bit SIMD. Like with AVX before it, the
number of SIMD registers is doubled to thirty-two and they’re widened so that
the old 256-bit registers are the low halves of the full 512-bit ymm registers. 

Additionally, there are eight new 64-bit mask registers. These are essentially
treated as vectors of Booleans. Operations which produce Booleans, such as
comparisons, now report their results in these registers.

Like before, there is a new encoding prefix, this time called EVEX. Also, like
before, it can also be applied to encode narrower SIMD, permitting all widths to
leverage the new registers.

AVX-512 is perhaps the most fragmented of x86’s SIMD families. A few slides ago,
you may have noted that it had far more entries than other families. To some
extent, this is because AVX-512 goes to great lengths to round out x86’s SIMD
capabilities, bring a much broader range of operations than was seen before. For
example, comparisons or floating-point conversions involving unsigned integers
were not present in any earlier extensions. Similarity, integer multiplications
and floating-point conversions involving 64-bit integers were also missing from
previous extensions.

AVX-512F is the foundation of AVX-512, and mostly focuses on processing 32 and
64-bit elements in 512-bit registers. The amount of new operations which it
introduces is much greater than I can do justice to. For 8 and 16-bit integers
you’ll need AVX-512BW, or byte and word. For more a more well-rounded set of
instructions for manipulating 32 and 64-bit integers in 512-bit registers, we
have the AVX-512DQ or double and quadword extension. If you want to use any of
AVX-512’s new operations on 128 and 256-bit wide registers you’ll need AVX-512VL
or vector length. AVX-IFMA52 is a rather curious extension which adds
fused-multiply add operations on 52-bit integers packed in 64-bit lanes. It
seems that this is fundamentally a way to leverage double-precision float
hardware for integer workflows.  AVX-512FP16 brings facilities for processing
half-precision floats, roughly equivalent to those which x86 already has for
single and double precision floats. Contrary to what you might expect from the
previous extension, AVX-512BF16 does not really add facilities for processing
brain floats, but instead it’s more about dot products specifically and
conversions between brain floats and single precision floats. Obviously, this is
meant for machine learning applications.

Even more obviously, AVX-512VNNI, or vector neural network instructions is even
more blatantly targeted at machine learning applications by accelerating integer
dot products.

A decent portion of AVX-512’s extensions are meant to accelerate bitwise
operations. This includes VBMI, VBMI2, BITALG, VPOPCNTDQ. Between these we get
population count instructions, double-wide shifts, bitwise rotations, and more.

AVX-512CD or conflict detection somewhat fits into the aforementioned group
because it adds leading zero count instructions for 32 and 64-bit elements, and
leading zero counts are conventionally thought of as bitwise operations. That
said, it’s mostly about identifying duplicate elements in a vector to more
elegantly handle certain edge cases. AVX-512VP2INTERSECT serves a similar
purpose but it’s rather obtuse about it to be frank. Intel has abandoned this
particular extension but AMD is keeping it around for now, which is the only
reason I mention it.

Although the AVX-512 family is quite large and diverse, the availability of most
extensions in consumer hardware is actually surprisingly consistent. This is
primarily because very few people have AVX-512.

### AVX-10 Family

AVX-10 is the planned future of SIMD on x86.

The large amount of extensions in the AVX-512 family means that there is a lot
of fragmentation .

AVX10.1 essentially establishes a common feature set by adopting most of the
AVX-512 family, essentially providing a foundation on which future SIMD
facilities can build off of. The idea is that future SIMD extensions will be
AVX10.2, .3, .4, so forth and so on.

AVX10.2 is the only one of those that is currently planned and it’s essentially
a giant toolkit for accelerating machine learning applications. This extension
essentially brings brain floats up to near feature parity with single/double
precision floats (with some caveats). It also brings even more integer dot
products, tiny float conversions, and really just a wide range of instructions
useful for ML.

I should note that if you were in the audience when I first delivered this talk,
you might notice that the content of this section is different, and this is
because Intel has changed their plans for AVX-10. The plan was previously that
AVX-10 was going to be implementable at either 256 or 512-bits wide. However, as
of this March, Intel has updated the AVX-10 technical paper to say that a
512-bit implementation is required.

## A Majority of x86?

The title of this talk alludes to SIMD comprising a majority of x86. I chose a
title which brings attention to this because in my experience talking to others,
they often find it surprising. It seems that a lot of programmers are under the
impression that SIMD is just some small part of the instruction set, that it's
just something that exists in its own little corner.

But this is really far from the case. As we’ve just seen, x86 has dozens of SIMD
extensions, which have been the primary method by which x86 has grown since the
late 90s, meaning that SIMD now dominates the ISA. The natural question to ask
at this point is, “To what extent?”.

In particular, I think it’s worth approaching this question pragmatically. By
this I mean we should consider only those instructions which a typical x86
programmer might potentially benefit from. Any instruction that an average
programmer would immediately write off as being irrelevant should be excluded.
This includes removed, deprecated, or outdated instructions, such as those from
MMX and x87. It also includes privileged instructions that a user-space
application would not be able use, and instructions which currently aren’t
available on consumer hardware, such as AMX and AVX-10.

## Instruction Mnemonics

My approach to answering this question was to take the instruction mnemonics
from the Intel’s Software Developer's Manual and manually categorized them as
either scalar, vector, or scalar within a vector register (recall what was
mentioned earlier about scalar floating-point operations using the first lane of
SIMD registers).

I should state upfront that I very likely made at least a few mistakes while
categorizing so it’s reasonable to assume these numbers aren’t entirely
accurate. Furthermore, it must be admitted that counting instruction mnemonics
is not necessarily a great metric to use here. CMOV has dozens of different
forms, while LEA has only three, yet they are counted the same. However, I think
these numbers are, if nothing else, useful for stimulating thought and
discussion.

Regardless, the result for the proportion of pure vector operations is a fairly
high 71%, and if we choose to include the scalar operations in registers into
the count, it’s an even greater 82%.

Even if we assume a massive margin of error, it’s safe to say that x86 has a lot
of SIMD in it.

## Applying SIMD

Now, the natural question to ask is how you can actually leverage SIMD
instructions in your programs.

An option that should come as no surprise would be to use assembly or inline
assembly. SIMD instructions are just instructions at the end of the day. This
approach naturally features all of the same strengths and weakness that you
typically associate with the use of assembly. A lot of precise control and the
the highest potential for performance but also requiring a larger investment of
time, and prerequisite knowledge. If this approach is of interest to you, you
can take a look through the second volume of Intel’s Software Developer’s Manual
for a list of instructions or one of the online resources which mirror this
document.

A second option, and the one that will be explored here is to use compiler
intrinsics. For our purposes, these are special functions that act as requests
for the compiler to emit a particular CPU instruction, or a short sequence
thereof. I say request because compiler optimizations still apply, so the use of
a specific intrinsic is not a guarantee that any particular instruction will
actually be emitted.

Another approach I’d like to mention is the use SIMD libraries, which
essentially provide a more convenient interface by creating thin abstractions
over compiler intrinsics and/or inline assembly. Personally, I’m not a big fan
of these, because they can only ever do a good job of abstracting over a small
subset of available instructions, meaning that you trade performance and control
for a cleaner interface. This tradeoff probably doesn’t make sense because if
you’re writing SIMD code, it’s probably because you’re trying to maximize
performance.

Another option would be to not really do anything at all, but instead lean on
compiler optimizations, namely auto-vectorization. Besides maybe adopting a
tendency to write code in a straightforward manner that is likely to be easy for
compilers to optimize it involves doing relatively little. I see it commonly
claimed that this is sufficient, but I would strongly disagree on this point.
There are many, many straightforward optimizations that modern compilers simply
miss with regards to SIMD.

## Compiling & Running

### Checking Your CPU's Features

Before starting to dabble with SIMD, you'll need to know which extensions your
CPU supports. Running a program that contains instructions which are not
supported by your processor will, unsurprisingly, raise an illegal instruction
exception.

Assuming you're on a Linux machine, the more straightforward way of doing this
would be to use the `lscpu` command and to inspect the listed flags.
Alternatively, you may inspect the result of catting the `/proc/cpuinfo` file,
which will contain much the same information, just on a per-core basis.

You can also use your compiler to query this info using the commands listed
here, which you might find useful on other operating systems.

Intel's SDE

Note that you don’t strictly need a processor which supports the extensions you
wish to play around with.

Intel's Software Development Emulator is a tool used to run programs having
instructions which your CPU may not support. Performance-wise it is course not
the same as running purely on hardware, but it enables you to explore new
instruction sets, and also test the correctness of your code on a wider range of
machines. For example, you may use it to test AVX-512 code on a CI machine that
doesn't necessarily have AVX-512.

### Compilation Flags

When compiling your program you’ll need to pass the pass the appropriate flags
to inform the compiler of which extensions it’s permitted to use.

For a list of relevant flags for GCC, which are also supported by Clang and ICP,
I would recommend looking up GCC’s x86 Options page online.

Now, there are technically a few different approaches to specifying CPU
features.

Perhaps the easiest approach, and the one that is most suitable when simply
dabbling with SIMD is the use the `-march=native` flag, which will mean that
your compiler will use the CPU it’s running on as the compilation target.

Alternatively, you may pass the name of a particular microarchitecture for the
`-march flag`, which will enable the feature set of that microarchitecture.

You can also specify individual CPU extensions, which is likely the best choice
for practical use. This avoiding enabling more CPU features than those which you
specifically intend to leverage, thereby avoiding accidentally imposing stricter
requirements than intended.

## Intel's Intrinsics Guide

For an actual list of intrinsics, the primary resource you’ll want to lean on is
Intel’s Intrinsics Guide. This acts as a convenient online resource that
documents x86’s various instrinsics.  These intriniscs are largely implemented
by GCC, Clang, MSVC, and ICPX, with only minor, irrelevant exceptions. This
means using intrinsics is fairly portable across mainstream x86 compilers.

As programmers, you all will be intimately familiar with the idea of
implementing more complex operations in terms of simpler operations. In doing
this, we typically treat the operations offered by our language and its standard
library as being our most primitive algorithmic building blocks. In my view,
learning to how write SIMD is fundamentally a matter of developing a familiarity
with a new set of primitive operations, and developing a fluency in their use,
much in the same manner that we all did when we first learned how to work with
ints, floats, if-statements, etc.

Naturally, this means first step to becoming proficient at working with x86 SIMD
is to become familiar with this new set of primitive operations. Since we’re
working with intrinsics here, this means reading through this guide and learning
what it has to offer. But that is easier said than done. Taking a look at the
page, it is initially rather esoteric. It uses unfamiliar data types, the
intrinsics follow an obtuse naming scheme, and there are thousands of entries
with no clear place to start exploring them all.

Despite being called a guide, it raises far more questions than it answers. For
that reason I’m going to try to explain some basic details of how to make sense
of this guide, I’m going to talk about some of the basic operations that are
going to be some of the first you’d want to acquaint yourself with, and I’m also
going to address a couple of practical challenges that you’re likely to
encounter early on.

### Header

The very first thing to do to write SIMD code using intrinsics is to include the
`immintrin.h` header. This will give you the declarations for all SIMD
intrinsics and the types that they use.  

### Data types

Those types themselves generally represent a SIMD register of a particular
width, packed with elements of a particular type, both indicated by the type’s
name.

The names begin with a pair of underscores followed by a letter m, which is
followed by a number indicating the width of the register. The name is then
closed by a zero to two letter suffix, which indicates the type of the elements.

A suffix of I indicates that the vector contains integers, regardless of how
wide those integers are, or whether they are unsigned or signed. When it comes
to floating-point elements, there are individual suffixes however. H for
half-precision, no suffix for single-precision,  D for double precision, and BH
for 16-bit brain floats.

A notable property of these types is that like `char`, and `std::byte` they are
permitted to alias arbitrary objects in memory. This detail is relevant in at
least one notable way discussed shortly.

There are also types which represent the mask registers introduced by AVX-512,
or rather some portion of their lower bits. Their names are more
straightforward, simply being __m mask, then their width. In practical terms,
they are consistently implemented as aliases to appropriate unsigned integer
types, much like how the fixed-width integer alias from the cstdint header are
defined. However, as far as I can tell, nothing about Intel’s documentation
suggests that this is required.

Now, we need to remember that there is a big difference between variables in a
programming language and the registers in our machines. Perhaps most notably, we
have only a small fixed number of registers, while we can declare a practically
unlimited number of variables of any of these types. If you have too many of
these variables alive at at given moment, as in from liveness analysis in
compilers, your compiler will begin to spill registers. This is naturally bad
for performance, so it’s a good idea to keep the number of these variables down.

### Intrinsic Naming Scheme

The intrinsic functions have their own obtuse naming scheme. They begin with
underscore mm. After that comes the width of the operation being performed,
except for when it’s 128 bits, in which case it’s excluded. The name of the
actual operation comes next and not surprisingly there’s lots of options there.
I’m going to try to cover some of them shortly. The intrinsic function names are
closed off by something that I am going to call a content type specifier.

These consist of two parts:

The first part is either S, P or EP. S indicates that the operation only
processes the first element in the vector, P indicates that the operation
processes all elements which are packed into the vector, and EP likely to stand
for extended packed and it’s just synonymous with P. It’s seems to only exist
because there would otherwise be naming conflicts between MMX intrinsics and SSE
intrinsics. Since MMX only had integer operations, these conflicts do not exist
for floating-point op, so their intrinsics simply use P while integer operation
intrinsics use EP.

The second part of the type specifiers indicates the type of the elements
themselves using a straightforward naming scheme. Floats get a one or two letter
abbreviation while integers get an I or S followed by their width.

I’d also like to mention out that sometimes the intrinsics are framed as
operating on vectors populated with a single integer taking up the entire
register, so sometimes you’ll see si128, si256, or si512 being used for the type
specifier. We’ll see this a couple of times later on.

## Operations

### Loads/Stores

Since the SIMD types represent registers you’ll find that you’re going to be
using them in a fashion that is somewhat comparable to the way that you might
work with registers in assembly. You’ll be performing explicit loads,
manipulating the data while it’s kept inside of these vector objects, and then
performing explicit stores back to memory.

For this purpose you use the load, loadu, store, and storeu intrinsics, which
perform the most basic SIMD load/store operations. These simply operate on a
contiguous sequence of bytes equal in size to the register being written to or
read from.

The operations that do not have a U suffix are aligned, meaning that the address
you pass to the operation has to be a multiple of the size of the block of data
being moved. Otherwise, a segmentation fault is produced. The operations that do
have U in the name are unaligned, and impose no such requirement.

The aligned operations are theoretically faster than their unaligned
counterparts, but in practice the difference is quite small. So long as they’re
passed aligned addresses, unaligned load and store operations will tend to
perform very comparably to their aligned counterparts.

It should be noted that the intrinsics for these operations introduced prior to
AVX-512 will take a pointer to the corresponding vector type when dealing with
integer data, because the operations are framed as a operating a single integer
the size of the entire register. The ability of the SIMD vector types to alias
arbitrary memory is quite useful here since the integer loads/store require some
pointer casting that would otherwise be UB. 

Since AVX-512 this has been simplified by having these intrinsics simply use
void pointers in all cases.

I should note that the tables, like this one, that I’ll be using to show the
intrinsics I’m talking about will usually have only list 128-bit forms since the
tables would be exceedingly large otherwise, but all of the intrinsics that I
will be talking about today also have 256-bit and 512-bit counterparts. 

### Broadcast Intrinsic

In the case that you wish to populate all lanes of a vector with a single value,
you can use the set1 intrinsics. This comes up a lot because when it comes to
SIMD we don’t have immediate values in the same way that we do in scalar code.
If you’re implementing a formula where you need to add three to some value,
you’re going to need to fill a SIMD vector with lots of threes.

This is a native operation with AVX-512, but with older instruction sets this
will tend to compile down to 2-4 instructions, depending on the size of the
value you’re broadcasting.

### Set Zero Intrinsic

In the very special case that you’re looking to fill a vector with all zeros,
there are the setzero intrinsics. These generally compile down to bitwise XORs
between a register and itself, which clears the content. An interesting detail
about this is that this is what’s known as a zeroing idiom. A device known as a
zeroing idioms units will recognize the use of this pattern within the
instruction stream and eliminate the need to schedule the instruction by
directly updating the register alias table. In effect, zeroing a register out,
is practically free. This goes for not just SIMD registers, but also
general-purpose registers, and AVX-512 mask registers.

### Floating-point Arithmetic

After populating your vectors, you’ll of course want to actually manipulate
their contents. 

We’ll start by taking a look at floating-point arithmetic where you'll find some
familiar operations: addition, subtraction, multiplication, and division.
There’s also the fused multiply-add instructions that were mentioned earlier.
You may even be pleased to see that there are square root operations and
approximate reciprocal instructions for 32-bit floats.

Probably the neatest operation is an approximate inverse square root for
single-precision floats. As you may have already figured out, it's a hardware
implementation of the well-known inverse square root hack commonly attributed to
Quake. Note that it's something that we've had available to us since SSE, 26
years ago. Knowing that on a modern CPU you can compute 16 approximation in just
4 cycles, that original software hack probably doesn't seem quite as handy as
before.

Since x86’s scalar float facilities come from the same extensions that brought
the vectorized versions, the vector versions behave in much the same fashion as
to what you are already accustomed to. There are no real surprises there.

### Integer Arithmetic

When it comes to integer operations, the situation is slightly more complex. We
have addition, and subtraction which behave in pretty much the way you would
expect. Although there are nominally only signed additions and subtractions, the
same operations obviously work for unsigned integers as well. I’ve denoted this
fact in the table below with an R meaning redundant.

Of course we also have multiplication, but there's more nuance here than you
might be used to. When we multiply two equally sized integers, the product that
is produced is up to twice as large as the inputs. For example, the product of
two 32-bit integers make require up to 64 bits to store. Mainstream programming
languages only tend to expose the low half of this full double-wide product. But
we could reasonably be interested in obtaining the high half of the product
instead or the double-wide product in its entirety. x86 has SIMD multiplications
for all three of these scenarios.

The mullo operations are used to compute a product’s lower half. Something to
note is that the lower half of a product is the same for both signed or unsigned
multiplications so the mullo operations can be applied in both cases.

There are also mulhi operations and these do come in unsigned and signed
variants because the high half of a product does differ.

Then there are mul operations which produce full double-wide products. However,
these operations only process every other pair of inputs, those in the even
indexed lanes. This is so the double-wide product can be stored in what was
previously two lanes. So, as an example, if you had two 128-bit vectors, each
containing four 32-bit integers, although there are four pairs of inputs, the
output would be a single 128-bit vector containing two 64-bit products.

When it comes to integer division, x86 unfortunately does not have any SIMD
instructions for this, which does present a bit of a challenge, one that will be
talked about in moment.

### Bitwise operations

You probably won’t be surprised to see that we have SIMD bitwise OR, XOR, and
ANDs. But you may be surprised to hear that we don’t have bitwise NOTs, despite
this being possibly the simplest algorithmic building blocks that we are used to
having available. We again have a situation where there is a very fundamental
operation which we do not have available to us.

We do have an operation called a ANDNOT, which is essentially just a bitwise
AND, with the first input operand negated. Although this probably comes across
as a bit of a weird operation, this turns out to be fairly useful for something
that will come up shortly.

### Shuffling

Although I won’t explore them in any meaningful depth here, I’d like to
acknowledge the existence of shuffles. The fact that SIMD registers can hold
multiple elements creates a need for a category of instructions that rearrange a
vector’s lanes.

These instructions are generally of great practical importance because, since
basic SIMD loads and stores operate on contiguous bytes, they’re most efficient
and well-suited for operating on arrays of primitive types. A SIMD load or store
on an array of integers, or floats typically makes a lot of sense. But contrast
this with the practical reality that we often deal with arrays of structs that
might have a mixture of integers, pointers, floats, and let’s not forget
padding. Loading 16 contiguous bytes from such as an array is very likely not a
particularly useful operation. 

For this reason, you will often need to move data around to get a vector full of
elements suitable for uniform processing, although it would of course be ideal
to lay data out in a series of homogeneous parallel arrays to avoid the need to
shuffle in the first place.

There are certain operations which move data in a specific pattern. For example,
interleaving the elements of two vectors, or performing a shift on each 128-bit
block with a byte-wise granularity.

But there are also ways to rearrange data in a completely free-form fashion.

### Shuffle/Permute Ops

The most powerful operations for rearranging data are what x86 calls shuffles or
permutations. One way of understanding these instructions would be that one
vector is treated as an array of elements, and the second vector is treated as a
list of indices into said vector. From this, you get another vector which
contains those values that were indicated by the indices. 

There are variants of these instructions which take the indices as in the form
of an immediate value instead of a separate vector, but they are of course, less
interesting than their counterparts that take a second vector since they allow
for a run time selection of elements.

The PSHUFB instruction, the intrinsics for which is shown here, is a notable
example of such an instruction because it was the first. Operating on 8-bit
lanes, it permits a lookup on table of 16 bytes. Since it’s from such an early
extension, it’s also incredibly widespread so it’s something that you can rely
on being available.

There are various other shuffle instructions that came from later extensions,
but the landscape thereof is incredibly fragmented and inconsistent, making it
practically impossible to given a terse summary thereof.

However, AVX-512 has made the situation significantly simpler in that we now
have shuffles, or in this case it calls them permutes, for all element types,
and for all vector widths. There are even variants which use the concatenation
of two vectors as the lookup table, so we work with tables twice the size of
what would otherwise be the case

### Comparisons

I’m going to talk about comparisons just because it makes it easier to talk
about branchless programming later.

SIMD comparisons are somewhat interesting in that we’re used to having them
produce a single Boolean value, but when performing comparisons between multiple
pairs of elements we naturally get multiple results.

The comparison instructions that were introduced by the SSE and AVX families
report their results using a SIMD register. For each lane in the output, a value
of 0 is produced for to indicate false or a value of -1 is produced to indicate
true. Note that most instructions which consume these masks only look at the
most significant bit within each lane however.

The intrinsics for comparison operations all contain CMP in the name, and
especially with earlier intrinsics, this will be followed by an abbreviation
indicating a specific comparison to perform.

Notable, floating-point comparisons from AVX onwards simply have CMP in their
name and the actual comparison which they test for is determined by an
additional enum operand, which becomes an immediate value in the resulting
instruction.

Things change a lot with AVX-512 because unlike previous comparisons, these
don’t use a SIMD register for the output because they can of course use the new
mask registers. Furthermore, we finally get comparisons for unsigned integers.
Somewhat strangely, there are intrinsics that both have the specific comparisons
in the name, and those that take an enum value as an additional operand. Both
are functionally equivalent, producing the same machine code.

### Other Operations to Look Into

As much as I would like to, it’s impossible for me to explain all the SIMD
operations which x86 offers within the relevant time frame. So I will instead
list in my slides some operations which you are likely to find useful in a
general-purpose context. Please pause the video if you’d like to read through
these, and look up their entries in the intrinsics guide.

### Typical Pattern of Use

Showing an actual example of SIMD code is probably long overdue at this point,
and so I’ll take this as an opportunity to also illustrate a common pattern used
when writing SIMD code. 

I do apologize for the poor formatting, but this is, unfortunately, slideware at
the end fo the day.

This code leverages AVX-512 to compute the pairwise absolute difference between
elements in two arrays of floats, storing the result back into the first array.
The basic idea here is that the data is processed in 512-bit blocks in SIMD
fashion, and any remaining elements are processed in a scalar fashion.

Computing the amount of floats which fit into a 512-bit vector upfront acts as a
convenience, which makes it easier to compute the number of elements which will
be processed in chunks. Modern compilers easily optimize that line into just a
bitwise AND.

The first loop’s body demonstrates the basic pattern of usage for SIMD
intrinsics. We first load data into vectors. Then we perform some computation,
here it’s just a subtraction and a subsequent absolute value operation. (That
specific intrinsic compiles down to a bitwise AND at it’s core. There’s no
specific x86 instruction for this.) But regardless we get the desired result,
which we then store back using an unaligned store.

The second loop is the scalar fallback that handles the remaining elements which
don’t manage to populate an entire vector.

I should probably note that there are a number of ways to handle trailing
elements that are potentially more efficient, but falling back to scalar code is
the simplest approach and one that is usable regardless of which SIMD feature
set you have available.

## Emulating Operations

One of the primary challenges that comes from writing SIMD code is that
sometimes you don’t have the operations, the primitive algorithmic building
blocks, that you’re used to building off of.

As I and the tables I’ve shared allude to, SIMD facilities on x86 are often very
fragmented, and incomplete. You may have noticed earlier how patchy the table
looked for multiplication operations. For example, 64-bit multiplication was not
natively available prior to AVX-512. Similarly there were no unsigned
comparisons prior to AVX-512, and we saw that the same was true for even a
simple bitwise NOT.

It is a bit of an inconvenient truth that not all of the scalar operations we
are accustomed to have SIMD counterparts, whether these are scalar instructions,
or the miscellaneous utility functions we all rely on.

But if you approach this from the mindset of looking at SIMD as a new set of
algorithmic building blocks, then these differences are, in some sense, a
natural thing to expect. Indeed, the opposite situation is also the case. There
are many SIMD instructions which have no scalar versions: min and max
operations, saturating additions and subtractions, various dot products, etc.

So then, we end up in a situation where operations like a bitwise NOT, need to
be expressed in terms of those operations that we do have. In other words, we
have to emulate them.

In my view, this is a very big part of what writing SIMD code actually is,
because it’s such a common practical concern. This is a time-consuming and often
counter-intuitive work.

If you’re writing SIMD code and you find that you need to perform some operation
which is not available in SIMD form, your first instinct is likely to fall back
to scalar code, but I would strongly discourage this. Moving each individual
element to a general-purpose register and back entails a lot of overhead, and it
give up the massive throughput benefits that SIMD is all about. In my
experience, it is incredibly rare that there is not some sequence of SIMD
instructions that will perform better than falling back to scalar code.

Keep in mind that SIMD offers substantial improvements in throughput. This means
there is a lot of leeway for an emulated operation to consist of multiple,
potentially higher-latency, instructions, and they might intuitively feel heavy
and slow, but they will still very often perform better than scalar code.

### Bitwise NOT

We can emulate a bitwise NOT with early instruction sets fairly easily by
leveraging that ANDNOT operation I mentioned earlier. The idea here is that if
we pass a vector full of 1 bits as the second operand, we effectively turn the
AND part of this operation into a no-op, which leaves just the NOT.

To get the vector full of ones, I use a common trick, which is to compare a
register against itself. Every lane is naturally equal to itself, and remember,
that these SIMD comparisons set all the bits in each lane to 1 to report true.

This is an example of a one’s idiom, similar to the previously mentioned concept
of a zero idiom. Ones idioms are not eliminated from the instruction stream, but
a dedicated ones idiom unit does eliminate the false dependency on the
register's previous contents, allowing the instruction to be scheduled earlier
that it otherwise might be.

### 64-bit Multiplication

We can implement 64-bit integer multiplication using the 32-bit multiplications
we talked about earlier, and that would look something like this.

It’s fundamentally just the multiplication algorithm we all learned in
elementary school. We’re just treating the 64-bit integer as consisting of two
digits in a 2^32 base.

Admittedly, this code is actually slower for bulk processing than scalar
multiplication, which does raises questions about the value of this
implementation..

### Emulation Isn’t Going to Be Perfect

We can’t always match the throughput of a particular scalar operation using SIMD
code, but that shouldn’t be discouraging. This is not necessarily a sign that
you shouldn’t use SIMD, or that you should fall back to scalar code, as counter
intuitive as that might be.

Chances are, your algorithms will consist primarily of operations which are
natively available and which do benefit from being vectorized. And therefore,
most of your algorithm will be sped up. It is only a small number of operations
which will be slower. This means that there will still be a net speedup from
SIMD vectorization most of the time. 

### Libraries as References

I previously mentioned the existence of SIMD libraries, and how they can be one
option for writing SIMD code. While I personally wouldn’t recommend their use
most of the time, I do think they’re quite useful for filling in the numerous
gaps that you’re likely to encounter because they often have their own
implementations of various missing operations. These are generally tested, and
optimized, something which is very time consuming to do yourself when your
primary concern is implementing an actual piece of functionality in a practical
application.

However, what you can do is use these libraries as references for how to emulate
missing operations, reimplementing their approaches as necessary. 

Most SIMD libraries tend to be fairly general-purpose. Here I’ve listed some
examples that I would consider worth looking into.

Sleef is a bit unique in that it’s not general-purpose, but instead focuses on
providing vectorized implementations of the various functions found within the
cmath header. It’s also one library that I actually would recommend using
directly because its design is well-suited for integration with code that is
primarily written using intrinsics, meaning it’s no real trade off of
performance or control.

## Branching

One of the most notable challenges you’ll encounter when writing SIMD code comes
in the form of branches. SIMD and branches are somewhat antithetical. SIMD is
all about applying the same operation to multiple inputs while branches are all
about applying different operations in response to differing inputs.

There are in fact branches that cannot be vectorized at all. Simply consider
anything that involves a syscall, writing to a global variable, or calling an
API. And of those branches that can be vectorized, most are too challenging and
inconvenient to do so in practice. However, there are a set of commonly
reoccurring tricks that can go a long way to help vectorize simple branches.

### Branchless Tricks

The simplest branches to vectorize are those that have semantics that are easily
expressed in some pure closed-form expression, essentially those which have a
functionality that you can recreate without using a branch.

### Conditional Arithmetic

As an example, conditional decrements and increments are straightforward with
SSE and AVX. Recall that the masks we get from comparisons have lanes which are
either 0 or -1. Subtracting the mask leads to a conditional increment and adding
the mask leads to a conditional decrement.

It’s also easy to make additions and subtractions conditional as well. A simple
bitwise AND between the addend and the mask is all that’s necessary, since
adding or subtracting zero has no effect of course.

### Conditional Bitwise Ops

Another simple trick is that you can perform a conditional bitwise NOT by using
a bitwise XOR between the mask and the value that you want to negate.

### Conditional Negation

By combining the tricks for conditional incrementation and for a conditional
bitwise NOT, we can implement another trick: a conditional two’s complement
negation. After all, that formula is nothing more than a bitwise NOT and an
increment. Making those individual operations conditional by applying the
aforementioned tricks makes the negation conditional as well.

### Blend

But the fundamental backbone of branchless programming is an operation which x86
refers to as a blend. This operation performs a selection between two values
using a third value as a control, on a per-lane basis. In some sense, it's
analogous to the ternary operator in C++, and functionally similar to the scalar
CMOV instruction. 

x86 has had dedicated SIMD blends since SSE4.1 making them widespread in the
current hardware landscape. But even without it, it's possible to emulate a
blend with just 3 instructions when working with earlier instruction sets.

The operation is important because in some sense, it allows you to make any
register-to-register operation conditional. All you have to do is blend between
the result of the operation and the original values fed into it, to limit that
operation’s effects to specific lanes.

### AVX-512 Masks

I’ve mentioned that if you have AVX-512, there is a new set of mask registers,
but something that I haven’t mentioned is that AVX-512 makes it trivial to make
any operation conditional.

Almost every operation which is encoded using AVX-512’s EVEX prefix can take a
mask register as an additional operand and use it in two different ways.

The first option is to perform a blend operation between the result of the
operation and another vector that the instruction will take in as an additional
input. The intrinsics which use this blend masking will have “mask” prefixed to
the name of the operation. 

The second option is to simply zero out the lanes which the mask marks as
inactive. The intrinsics for using this zero masking will have “maskz” prefixed
to the name of their operation instead.

### If-Statements

When talking about branches, if-statement are probably the first thing to come
to mind. While not all if-statements can be vectorized, a decently wide range of
simple if statements can be vectorized somewhat easily and in a somewhat
systematic fashion. This is especially the case for if-elses that looks
something like this.

In particular, note that the two blocks of conditional code are simply assigning
to some variables that exist outside of the scope of the if-statement itself.
I’m going to assume that the expressions here can be computed without external
side effects and that this means the individual expressions can be vectorized in
a vertical fashion.

I’m going to be using this if-else as bit of a toy example for how you might go
about transforming a simple branch into code that is more easily vectorizable.

The very first thing you will want to do is to factor out any common
sub-expressions between the two branches, since these can be done
unconditionally. In this example, we have the call to foo being factored out and
assigned to t0.

We can also do the same to the calls to bar even though they are being passed
different arguments because we can easily select between the differing values
using a blend operation. Hence, we get the variable t1.

As this point, we can apply one of our branchless tricks to expand the range of
operations which we can consider equivalent. We can note that one path features
an addition while the other features a subtraction, both on the same expression.
By using the conditional negation trick that was mentioned earlier, we can
factor our yet another term, and replace the original subtraction with an
addition.

This gets the two expressions being assigned to the variables to be equivalent,
meaning we can factor them out completely as well.

At this point we can detach the else clause by turning it into a separate
if-statement, predicated on the negation of the original condition.

And the very last transformation we’ll do to this code is to replace these
simplified if-statements with blend operations. These will select between each
variable’s original value and their new value, then assign the result
unconditionally.

This new code is now entirely in terms of operations which can be
straightforwardly vectorized by simply replacing the scalar operations with
their SIMD counterparts.

I would like to point out that these kinds of transformations may also be
applied to while loops since they are essentially just repeated if statements.
So this kind of thinking not only permits for simple if-statements to be
vectorized, but simple loops as well.

### Partitioning/Filtering

When the computations we’re performing are not trivial and when we have to
process more elements than one or two vectors can hold, the previous approach
isn’t necessarily optimal. All of the work done by the lanes which are masked
out is essentially lost potential.

In such a case it’s generally beneficial to first coalesce elements based on
which branch they need to go down so they are adjacent in memory.

For an if-statement, this generally means filtering the data, which you might do
with something to the effect of std::copy_if.

If it’s an if-else, then you’ll want to ensure the data is partition the data
instead, which of course can be done with std::partition.

Either of these approaches essentially transforms the necessary processing into
the simpler case of processing an array (or two) in a uniform manner.

Assuming the necessary processing is heavy enough, the cost of
filtering/partitioning the data upfront will be more than offset out by
throughput increases of the SIMD vectorization, leading to an overall speedup.

## Closing Thoughts on SIMD

More than fifty years ago, we got the C programming language, and to this day,
it's considered to be one the best languages for working close to the hardware
and for achieving high performance. This is still no programming language which
is able to consistently achieve better performance across a wide range of
applications. I don’t think this is because we’re hitting the limits of our
machine’s potential, but because current languages are not enabling us to
leverage the advancements that ISAs have made over the past few decades.

The landscape of x86 SIMD serves to demonstrate that ISAs have advanced greatly
since C was invented. Although this talk is primarily about x86, SIMD now
comprises a significant portion of every mainstream ISA and is widely available
on almost every kind of CPU you can imagine. I think SIMD is in some sense, the
biggest example of how current languages have not kept up with the new features
that ISAs now offer, but it’s also far from the only example. Scalar facilities
have also seen many substantial improvements in the same time period.

When we choose to use a programming language like C or C++, the principle
motivators are typically their superior control over the hardware and potential
for performance when compared to the vast majority of their competitors. But the
notion that these languages offer low-level control and are good for writing
fast code is challenged by the realization that they offer no facilities
corresponding to very large segments of basically every mainstream instruction
set in mainstream use today.

SIMD is often thought about as a fairly specialized domain, and it’s indeed true
that many SIMD instructions are designed to accelerate fairly specific
workflows, but it’s also the case that SIMD has a lot of potential in
general-purpose contexts. Just think about the instructions which I previously
mentioned. They were loads, store, arithmetic and bitwise operations. These are
fundamentally general-purpose operations. Even as far back as MMX, the designers
of SIMD instructions sets recognized their potential for accelerating more
general-purpose applications.

For these reasons, I think it would be more than appropriate to have a
programming language which treats SIMD and all the other advancements that have
been made along side it, as a core part of the language’s facilities, fully
integrated into its design, to the same extent that things like integer and
floating-point arithmetic are. I don’t know exactly what that might look like,
but I do know it’s something that I’d like to see.

Thank You.