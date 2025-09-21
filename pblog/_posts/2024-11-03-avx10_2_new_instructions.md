---
layout: pblog_post
title: AVX-10.2's New Instructions
description: A quick overview of the functionality of the new instructions introduced by AVX-10.2.
---

This is an overview of some of the information provided by Intel's AVX-10.2
Architecture Specification which is currently available at the following link:

[https://www.intel.com/content/www/us/en/content-details/836199/intel-advanced-vector-extensions-10-2-intel-avx10-2-architecture-specification.html](https://www.intel.com/content/www/us/en/content-details/836199/intel-advanced-vector-extensions-10-2-intel-avx10-2-architecture-specification.html)

The focus is on instructions that perform fundamentally new operations, or
changes which appreciably change how you might exploit SIMD instructions. Some
of AVX-10.2's more minor additions have been left out because they are
fundamentally similar to existing instructions.

## EVEX Encoded 256-bit SIMD
AVX-10.2 allows for 256-bit wide SIMD instructions to be encoded using an
extended EVEX prefix. This permits 256-bit operations to have embedded rounding
modes/suppress all exceptions control, a feature that was introduced with
AVX-512 but limited to 512-bit wide instructions. It also permits them to access
the increased total of 32 vector registers and the vector mask registers that
were originally introduced by AVX-512F. 256-bit instructions that could do the
two last things were technically present with AVX-512VL, but now this is
possible available without that extension.

## Zero-Extending Moves to Vector Registers
Two new instructions, `vmovw` and `vmovd` facilitate the common practice of
moving a 16 or 32-bit element from memory into the first lane of an XMM register
while zeroing out the remaining lanes. The 16 or 32-bit value does not need to
come from memory, and can instead come from another XMM register, making it a
bit terser to copy the first lane from one vector to another.

WHile, not the most exciting things in the world, they are bound to help shave a
few cycles off our execution times and a few bytes off our executable sizes.

## Double-wide Single-Precision to Half-Precision
`vcvt2ps2phx` converts two vectors of single-precision floats to a single vector
of half-precision floats. Functionally, this is just a more efficient
alternative to the `vcvtps2ph` instruction that we got from AVX-512F and F16C
since it operates on twice as many inputs.

## Brain Float 16 Instructions
AVX-10.2 introduces instructions for manipulating brain floats.

A brain float is 16-bit floating-point format with an 8-bit exponent field and a
7-bit mantissa field. This is in contrast to IEEE-754 16-bit floats which have a
5-bit exponent field and a 10-bit mantissa field. 

The reason for dedicating more bits to the exponent field is to match the
dynamic range of 32-bit floats, dynamic range here being the ratio of the
largest and smallest representable positive values. The brain float format was
created for use in machine learning applications where dynamic range is a more
important property than the mantissa's resolution. Being half the size of the
common single-precision float, they naturally have the half the memory
footprint, can loaded and stored at twice the rate, and have smaller, simpler
hardware implementations.

Some of these instructions will be familiar to anyone who has delved into x86's
existing floating-point instructions:
* `vaddnepbf16` - addition
* `vsubnepbf16` - subtraction
* `vmulnepbf16` - multiplication
* `vdivnepbf16` - division
* `vrcppbf16` - approximate reciprocal
* `vmadd***nepbf16` - fused multiply-add
* `vmsub***nepbf16` - fused multiply-subtract
* `vmnadd***nepbf16` - fused negated multiply-add
* `vmnsub***nepbf16` - fused negated multiply-subtract
* `vsqrtnepbf16` - square root
* `vrsqrtnepbf16` - approximate reciprocal square root
* `vcomsbf16` - scalar comparison
* `vcmppbf16` - vector comparison
* `vmaxpbf16` - maximum
* `vminpbf16` - minimum
* `vdpphps`- dot product

Additionally, there are also brain float instructions that perform operations
you may not be familiar if you haven't explored the new instructions that
AVX-512 brought.

* `vfpclasspbf16` - Produces a mask based on whether the input value belongs to
  any of the floating-point categories indicates by an immediate value.
* `vgetexppbf16` - Effectively computes `floor(log2(x))`. Note that unlike
  manually extracting the exponent field the result of this instruction is a
  float, and it properly handles denormal values.
* `vgetmantpbf16` - Extracts the significand of the input float.
* `vreducepbf16` - Essentially compute the remainder of floating-point division
  by a power-of-two constant between `2^0` and `2^-15`. The result is the
  abstract value `n - round(n / d) * d` where the rounding operation can be any
  of x86's four floating-point rounding modes.
* `vrndscalebf16` - Perform a rounding operation to an amount of fractional bits
  ranging in [0, 15]. The rounding scheme can be any of the four floating-point
  rounding schemes encodable by the MXCSR.RC bits.
* `vscalebf16` - Multiplies the first operand by 2 raised to the power of the
  whole part of the second operand.

You may have noticed that many of these instructions have an unfamiliar `ne`
infix in their name. I haven't been able to find documentation that definitively
confirms its meaning, but it seems to stand for "nearest", in reference to the
round-to-nearest rounding mode. This is because these instructions always use
the round-to-nearest strategy. In fact, they ignore current state of the MXCSR
register altogether. They always behave as if flush-to-zero and
denormals-as-zero are both enabled and they also do not raise floating-point
exceptions. Presumably, these decisions were made to simplify the implementation
of these instructions, making it easier to achieve favorable performance
characteristics.

## Extended Scalar Floating-point Comparisons
Some of the new instructions are scalar floating-point comparisons:

* `vcomxsh` - perform comparison on scalar half-precision floats
* `vcomxss` - perform comparison on scalar single-precision floats
* `vcomxsd` - perform comparison on scalar double-precision floats
* `vucomxsh` - perform unordered comparison on scalar half-precision floats
* `vucomxss` - perform unordered comparison on scalar single-precision floats
* `vucomxsd` - perform unordered comparison on scalar double-precision floats

x86 has had scalar floating-point comparisons for a long time. There are older
counterparts: `vcomish`, `comiss`, `comisd`, `vucomish`, `ucomiss`, and
`ucomisd` respectively. You may note that the new instructions have an `x` in
their names, which denotes them as the extended versions. The fact that they're
called "extended" might suggest that they have an additional functionality, but
actually the only thing that is extended is the mechanism by which they report
their results.

For those who may not already be aware, these instructions don't specifically
test if two floats compare in any particular fashion. Instead, they set flags
within the EFLAGS register in patterns that indicate what relationship exists
between the two floating-point inputs. After these flags are set, an instruction
that reads them such as `jCC`, `setCC` or `cmovCC` is used. The `CC` is a
placeholder for the abbreviated name of a condition code, that is, effectively a
particular pattern that the EFLAGS register must be in.

Below are all of x86's condition codes, their abbreviations, and most
importantly, the actual condition they test for:

| Abbrev.: | Name:                | Condition:         |
|----------|----------------------|--------------------|
| A        | Above                | CF = 0 and ZF = 0  |
| AE       | Above or equal       | CF = 0             |
| B        | Below                | CF = 1             |
| BE       | Below or equal       | CF = 1 or ZF = 1   |
| C        | Carry                | CF = 1             |
| E        | Equal                | ZF = 1             |
| G        | Greater              | ZF = 0 and SF = OF |
| GE       | Greater or equal     | SF = OF            |
| L        | Less                 | SF != OF           |
| LE       | Less or equal        | ZF = 1 or SF != OF |
| NA       | Not above            | CF = 1 or ZF = 1   |
| NAE      | Not above or equal   | CF = 1             |
| NB       | Not below            | CF = 0             |
| NBE      | Not below or equal   | CF = 0 and ZF = 0  |
| NC       | Not carry            | CF = 0             |
| NE       | Not equal            | ZF = 0             |
| NG       | Not greater          | ZF = 1 or SF != OF |
| NGE      | Not greater or equal | SF != OF           |
| NL       | Not less             | SF == OF           |
| NLE      | Not less or equal    | ZF = 0 and SF = OF |
| NO       | Not overflow         | OF = 0             |
| NP       | Not parity           | PF = 0             |
| NS       | Not sign             | SF = 0             |
| NZ       | Not zero             | ZF = 0             |
| O        | Overflow             | OF = 1             |
| P        | Parity               | PF = 1             |
| PE       | Parity even          | PF = 1             |
| PO       | Parity odd           | PF = 0             |
| S        | Sign                 | SF = 1             |
| Z        | Zero                 | ZF = 1             |
{: .table.wide .table.align_left  }


Some of these condition codes have names that suggest a relationship to
comparisons, such as Equal or Not equal. A point that may be slightly confusing
to those who are not already familiar with these is the simultaneous presence of
codes called Above & Greater or Below & Less, as it's not immediately clear how
they differ. Those codes that use Above and Below are meant to be used when
working with unsigned integers, while those codes that use Greater and Less are
meant to be used when working with signed integers. It will come as no surprise
that the Equal and Not Equal condition codes may be applied to both unsigned and
signed integers.

But which condition codes do you use when comparing floats?

Things get more subtle there and some more background information is necessary
to make sense of the situation. It's been mentioned that these comparison
instructions report their results through the EFLAGS registers. When it comes to
the older floating-point comparisons, they set the ZF, PF, CF flags based on the
relationship that exists between the two inputs. The AF, OF, SF flags are
unconditionally set to zero. The patterns are set as follows:

| Comparison:  | AF | OF | SF | ZF | PF | CF |
|--------------|----|----|----|----|----|----|
| unordered    | 0  | 0  | 0  | 1  | 1  | 1  |
| less-than    | 0  | 0  | 0  | 0  | 0  | 1  |
| equal        | 0  | 0  | 0  | 1  | 0  | 0  |
| greater-than | 0  | 0  | 0  | 0  | 0  | 0  |
{: .table.narrow .table_align_left_first_column }

If we look at how this pattern of setting these flags interacts with the
relevant condition codes mentioned earlier, we get the following table:

| Name:                | Unordered: | Less | Equal | Greater |
|----------------------|------------|------|-------|---------|
| Above                | 0          | 0    | 0     | 1       |
| Above or equal       | 0          | 0    | 1     | 1       |
| Below                | 1          | 1    | 0     | 0       |
| Below or equal       | 1          | 1    | 1     | 0       |
| Equal                | 1          | 0    | 1     | 0       |
| Greater              | 0          | 1    | 0     | 1       |
| Greater or equal     | 1          | 1    | 1     | 1       |
| Less                 | 0          | 0    | 0     | 0       |
| Less or equal        | 1          | 0    | 1     | 0       |
| Not above            | 1          | 1    | 1     | 0       |
| Not above or equal   | 1          | 1    | 0     | 0       |
| Not below            | 0          | 0    | 1     | 1       |
| Not below or equal   | 0          | 0    | 0     | 1       |
| Not equal            | 0          | 1    | 0     | 1       |
| Not greater          | 1          | 0    | 1     | 0       |
| Not greater or equal | 0          | 0    | 0     | 0       |
| Not less             | 1          | 1    | 1     | 1       |
| Not less or equal    | 0          | 1    | 0     | 1       |
{: .table.wide .table.align_left  }

You can go through this table row-by-row and see where it does and doesn't line
up with your intuitions and expectations, but I think a method of interpreting
this information that requires less active thought on the part of the person
casually reading this would be to work backwards by trying to map the rows onto
what we would expect from the comparison operators in mainstream programming
languages:

| Operator | Behavior | Matches      |
|----------|----------|--------------|
| <        | 0 1 0 0  | -            |
| <=       | 0 1 1 0  | -            |
| >        | 0 0 0 1  | A, NBE&nbsp;&nbsp;&nbsp;&nbsp; |
| >=       | 0 0 1 1  | AE, NB       |
| ==       | 0 0 1 0  | -            |
| !=       | 1 1 0 1  | -            |
{: .table.align_left  }

A few things may stand out here. First, most comparisons operators have no
corresponding condition code, and in fact only two do. Second, not all of the
condition codes that relate to comparisons fit into this table. If we construct
a table but for cases where the operators handle unordered relationships in the
opposite fashion, more condition codes are given a place:

| Operator | Behavior | Matches    |
|----------|----------|------------|
| <        | 1 1 0 0  | B, NAE     |
| <=       | 1 1 1 0  | BE, NA     |
| >        | 1 0 0 1  | -          |
| >=       | 1 0 1 1  | -          |
| ==       | 1 0 1 0  | E, LE, NG  |
| !=       | 0 1 0 1  | NE, NLE, G |
{: .table.align_left  }

However, even then, there are still condition codes that don't fit into either
table: L, NGE, GE, and NL. The first two always produce false, and the last two
always produce true, and therefore are of no real practical value.

If we wish to test for a greater-than or greater-than-or-equal relationship, we
can just use the Above and Above-Equal condition codes. If we wish to test for
less-than or less-than-or-equal, there are no dedicated condition codes, but
it's easy to work around by swapping the order of the operands to the comparison
instruction and using the Above and Above-Equal condition codes instead. The
real problem is that testing for inequality and equality is surprisingly
difficult since there are no corresponding condition codes.

Indeed, if you look at the code emitted by mainstream C compilers [(Example on
Compiler Explorer)](https://godbolt.org/z/ceoKhMxfY) when comparing two floats,
you'll note that it's a couple of instructions longer when comparing for
equality or inequality. It should be noted that this is not always the case and
depending on how the comparison result is being used, compilers may be able to
emit code that's the same length.

We can argue that this is the major shortcoming of the old floating-point scalar
comparisons, and it's something which AVX-10.2's new instructions addresses.

The new extended floating-point comparisons use the ZF, PF, CD flags to report a
relationship just like the old ones do, but they also use the OF and SF flags.
The AF flag is still set to 0 unconditionally however:

| Comparison:  | AF | OF | SF | ZF | PF | CF |
|--------------|----|----|----|----|----|----|
| unordered    | 0  | 1  | 1  | 0  | 1  | 1  |
| less-than    | 0  | 1  | 0  | 0  | 0  | 1  |
| equal        | 0  | 1  | 1  | 1  | 0  | 0  |
| greater-than | 0  | 0  | 0  | 0  | 0  | 0  |
{: .table.narrow .table_align_left_first_column }

Creating a table of interactions with condition codes as was done previously, we
get:

| Name:                | Unordered: | Less | Equal | Greater |
|----------------------|------------|------|-------|---------|
| Above                | 0          | 0    | 0     | 1       |
| Above or equal       | 0          | 0    | 1     | 1       |
| Below                | 1          | 1    | 0     | 0       |
| Below or equal       | 1          | 1    | 1     | 0       |
| Equal                | 0          | 0    | 1     | 0       |
| Greater              | 1          | 0    | 0     | 1       |
| Greater or equal     | 1          | 0    | 1     | 1       |
| Less                 | 0          | 1    | 0     | 0       |
| Less or equal        | 0          | 1    | 1     | 0       |
| Not above            | 1          | 1    | 1     | 0       |
| Not above or equal   | 1          | 1    | 0     | 0       |
| Not below            | 0          | 0    | 1     | 1       |
| Not below or equal   | 0          | 0    | 0     | 1       |
| Not equal            | 1          | 1    | 0     | 1       |
| Not greater          | 0          | 1    | 1     | 0       |
| Not greater or equal | 0          | 1    | 0     | 0       |
| Not less             | 1          | 0    | 1     | 1       |
| Not less or equal    | 1          | 0    | 0     | 1       |
{: .table.wide .table.align_left  }

Again, let's fill out a table for our preferred programming language's
comparison operators.

| Operator | Behavior | Matches |
|----------|----------|---------|
| <        | 0 1 0 0  | L, NGE  |
| <=       | 0 1 1 0  | LE, NG  |
| >        | 0 0 0 1  | A, NBE  |
| >=       | 0 0 1 1  | AE, NB  |
| ==       | 0 0 1 0  | E       |
| !=       | 1 1 0 1  | NE      |
{: .table.align_left  }

And if we reverse how unordered relationships are handled:

| Operator | Behavior | Matches |
|----------|----------|---------|
| <        | 1 1 0 0  | B, NAE  |
| <=       | 1 1 1 0  | BE, NA  |
| >        | 1 0 0 1  | G, NLE  |
| >=       | 1 0 1 1  | GE, NL  |
| ==       | 1 0 1 0  | -       |
| !=       | 0 1 0 1  | -       |
{: .table.align_left  }

At a casual glance, these tables look much cleaner than the old ones. You may
now note that the E and NE condition codes now correspond to the equality and
inequality comparison operators. Effectively this addresses the earlier issues
where compilers had to emit more instructions when comparing for equality and
inequality. Additionally, it's also just nicer to work with the condition codes
when performing other comparisons since you can simply use L, LE, A and AE.

The net effect is a small potential performance improvement and slightly
improved ergonomics.

## IEEE-754 2019 Min & Max Instructions
AVX-10.2 adds a set of new instructions for performing minimum and maximum
operations on pairs of floating-point numbers: 

* `vminmaxsh` - minimum/maximum operation on scalar half-precision float
* `vminmaxss` - minimum/maximum operation on scalar single-precision float
* `vminmaxsd` - minimum/maximum operation on scalar double-precision float
* `vminmaxph` - minimum/maximum operation on packed half-precision floats
* `vminmaxps` - minimum/maximum operation on packed single-precision floats
* `vminmaxpd` - minimum/maximum operation on packed double-precision floats
* `vminmaxnepbf16` - minimum/maximum operation on packed brain floats with
  rounding-to-nearest

Now, x86 has had instructions such as `minps` and `maxps` since SSE, and it also
got the `vreduce**` instructions with AVX-512F, which are also used to perform
minimum and maximum operations. With AVX-10.2 throwing its hat into the ring,
x86 now has three sets of instructions for performing minimum and maximum
operations. This naturally raises the question of what their differences are,
especially when finding the minimum or maximum of two numbers does not
intuitively come across as problem with a lot of nuance.

### SSE Min & Max
The oldest instructions for min and max operations are functionally equivalent
to the following pseudo-code:

<div>
  <code style="width: 49%; float:left;">min(x, y):<br>if x < y:<br>&nbsp;&nbsp;&nbsp;&nbsp;return x<br>else:<br>&nbsp;&nbsp;&nbsp;&nbsp;return y</code>
  <code style="width: 49%; float:right;">max(x, y):<br>if x > y:<br>&nbsp;&nbsp;&nbsp;&nbsp;return x<br>else:<br>&nbsp;&nbsp;&nbsp;&nbsp;return y</code>
</div>

Presumably these semantics were chosen to make it easier to translate what is
likely a common, but naïve, implementation of min/max operations into machine
code.

While this logic is fine under most circumstances, the function is asymmetrical
in a few subtle ways. Since less-than and greater-than comparisons against NaN
always yield false in mainstream programming languages, `min(1.0, NaN) = NaN`
while `min(NaN 1.0) = 1.0`. Additionally, `min(+0.0, -0.0) = -0.0` while
`min(-0.0, +0.0) = +0.0`. You can replace all the zeros with NaNs in that last
pair of expressions and they would also hold true. 

In effect, there is an order dependence on the inputs which would likely be
annoying and unexpected to programmers who have not been exposed to these
instructions before. 

### AVX-512 Range
AVX-512F introduced the `vrange**` instructions. Although the name may not
immediately suggest it, they're used to find mins and maxes, with a few optional
twists. The name refers to the relevance of max and min operations to performing
range restriction operations, i.e. clamping. 

These instructions, take an immediate value as their third operand and it
controls two details of how these instructions behave. The two low bits are used
to select whether the instructions should compute the minimum, maximum, minimum
of absolute values, or maximum of absolute values. The next two bits determine
how the sign bit of the result is computed. It can be copied from the first
operand, left unaltered, unconditionally cleared, or unconditionally set.

On top of this additional flexibility, the reduce operation is much better about
having symmetrical behavior. In the case where one input is NaN, it selects the
non-NaN value, i.e `min(1.0, NaN) = 1.0` and `min(NaN 1.0) = 1.0`. Additionally,
when comparing two zeros with different signs, the negative one is treated as
being less than than the one with a positive sign i.e negative zero is preferred
when computing the min and positive zero is preferred when computing the max.
This is also the case when comparing two NaNs with different signs. Therefore
this instruction addresses all of the aforementioned asymmetries that the SSE
min and max instruction had.

### AVX-10.2 & IEEE-754 2019 Minimum & Maximum Operations
AVX-10.2's `vminmax**` instructions are designed to follow the IEEE-754 2019
standard, which defines a total of eight different minimum and maximum
operations.

Like the `vrange**` instructions, the `vminmax**` instructions take an immediate
value which controls how the sign bit is computed and also controls which
operation is performed. However, this time, there are eight operations to choose
from, the eight defined by the IEEE-754 standard. The control over the sign bit
is the same, with your choice of copied from the first operand, left unaltered,
unconditionally cleared, or unconditionally set.

#### Minimum and Maximum
The minimum and maximum operations are the simplest.

If the first operand compares less/greater than the second operation, then the
first operand is chosen as the minimum/maximum respectively. If the first
operand compares greater/less instead, then the second operand is returned.
Negative zero compares as being less than positive zero. When the inputs are
otherwise equal, either is returned. Additionally, if one of the inputs is NaN,
a quiet NaN is produced.

#### Minimum Magnitude and Maximum Magnitude
The minimum magnitude and maximum magnitude number are slight variations where
the sign bits on the floating-point inputs are cleared for the purpose of
comparison. i.e. it's the absolute values of the numbers, their magnitudes,
which are compared.

#### Minimum Number and Maximum Number
The minimum number and maximum number operations handle NaNs differently than
the minimum and number operations. If only one of the inputs is a NaN, then the
non-NaN value is consider the min/max.

#### Minimum Magnitude Number and Maximum Magnitude Number
The minimum magnitude number and maximum magnitude number operations also
cleared the sign bits on the floating-point inputs for the purposes of
comparison.

## Saturating Floating-point to Integer Conversions
A large number of AVX-10.2's new instructions are conversions from
floating-point types to integral types which feature saturating behavior.
Saturation means that if a quantity is too large to be represented in the target
format, then the quantity represented is clamped to the nearest representable
value in the target format.

Existing conversion instructions have taken the approach of producing special
values or raising floating-point exceptions. For example, `cvttss2si` and
`cvtss2si` produce `0x80000000` for 32-bit operands when the input is too large
in magnitude for signed 32-bit integers, when the input is infinity, or when the
input is NaN.

These new instructions also generally come in truncating and non-truncating
forms. This is a pattern that should be familiar from existing conversion
instructions since it dates back to SSE. The truncating forms of these
instructions remove all fractional bits from the input when performing the
conversion, effectively rounding towards zero. The non-truncating forms of these
instructions default to using the current rounding mode to determine how
fractional bits are handled.

The new conversion instructions are as follows:

* `vcvttss2sis` - Convert a single-precision float to a 32/64-bit signed int
  with truncation and saturation.
* `vcvttss2usis` - Convert a single-precision float to a 32/64-bit unsigned int
  with truncation and saturation.

<hr>

* `vcvttsd2sis` - Convert a double-precision float to a 32/64-bit signed int
  with truncation and saturation.
* `vcvttsd2usis` - Convert a double-precision float to a 32/64-bit unsigned int
  with truncation and saturation.

<hr>

* `vcvtph2ibs` - Convert half-precision floats to signed 8-bit ints with saturation
* `vcvtph2iubs` - Convert half-precision floats to unsigned 8-bit ints with saturation
* `vcvttph2ibs` - Convert half-precision floats to signed 8-bit ints with
  truncation and saturation
* `vcvttph2iubs` - Convert half-precision floats to unsigned 8-bit ints with
  truncation and saturation

<hr>

* `vcvttps2dqs` - Convert single-precision floats to signed 32-bit ints with
  truncation and saturation
* `vcvttps2qqs` - Convert single-precision floats to signed 64-bit ints with
  truncation and saturation
* `vcvttps2udqs` - Convert single-precision floats to unsigned 32-bit ints with
  truncation and saturation
* `vcvttps2uqqs` - Convert single-precision floats to unsigned 64-bit ints with
  truncation and saturation

<hr>

* `vcvtps2ibs` - Convert single-precision floats to signed 8-bit ints with
  saturation.
* `vcvtps2iubs` - Convert single-precision floats to unsigned 8-bit ints with
  saturation.
* `vcvttps2ibs` - Convert single-precision floats to signed 8-bit ints with
  truncation and saturation.
* `vcvttps2iubs` - Convert single-precision floats to unsigned 8-bit ints with
  truncation and saturation.

<hr>

* `vcvttpd2dqs` - Convert double-precision floats to signed 32-bit ints with
  truncation and saturation
* `vcvttpd2qqs` - Convert double-precision floats to signed 64-bit ints with
  truncation and saturation
* `vcvttpd2udqs` - Convert double-precision floats to unsigned 32-bit ints with
  truncation and saturation
* `vcvttpd2uqqs` - Convert double-precision float to unsigned 64-bit ints with
  truncation and saturation

In addition to the previously listed conversions, there are also new
counterparts for brain floats. These follow the trend of always using
rounding-to-nearest where the current rounding mode would be used, and of never
raising floating-point exceptions.

* `vcvtnebf162ibs` - Convert brain floats to signed 8-bit ints with
  rounding-to-nearest and saturation.
* `vcvtnebf162iubs` - Convert brain floats to unsigned 8-bit ints with
  rounding-to-nearest and saturation.
* `vcvttnebf162ibs` - Convert brain floats to signed 8-bit ints with truncation
  and saturation.
* `vcvttnebf162iubs` - Convert brain floats to unsigned 8-bit ints with
  truncation and saturation.

## Tiny Float Conversions
In addition to the aforementioned brain float format, there also exist two 8-bit
formats that have been designed for machine learning applications. Theses are
called E4M3 and E5M2. The names refer directly the width of the exponent and
mantissa fields.

In response to a qtrend we're all aware of, Intel has added a number of
instructions for utilizing these instructions, mainly conversions.

### Conversions from Tiny Floats:
The presence of conversions from the E4M3 format is probably not surprising.
Curiously, there is no instruction to convert E5M2 floats to other formats
however.

* `vcvthf82ph` - Converts E4M3 floats to half-precision floats.

### Conversions to Tiny Floats
The majority of the operations involving tiny floats are conversions from
half-precision floats to these smaller formats.

* `vcvtneph2bf8` - Convert half-precision floats to E5M2 floats
* `vcvtneph2bf8s` - Convert half-precision floats to E5M2 floats with saturation
* `vcvtneph2hf8` - Convert half-precision floats to E4M3 floats
* `vcvtneph2hf8s` - Convert half-precision floats to E4M3 floats with saturation

<hr>

* `vcvtne2ph2bf8` - Convert two vectors of half-precision floats to a single
  vector of E5M2 floats.
* `vcvtne2ph2bf8s` - Convert two vectors of half-precision floats to a single
  vector of E5M2 floats with saturation.
* `vcvtne2ph2hf8` - Convert two vectors of half-precision floats to a single
  vector of E4M3 floats.
* `vcvtne2ph2hf8s` - Convert two vectors of half-precision floats to a single
  vector of E4M3 floats with saturation.

### Conversions to Tiny Floats w/ Offset
Some of the new instructions convert half-precision floats to E4M3 or E5M2
floats with an additional bias added to the input. These also come in forms that
exhibit saturation and forms that don't.

* `vcvtbiasph2bf8` - Convert half-precision floats to E5M2 floats with bias
* `vcvtbiasph2bf8s` - Convert half-precision floats to E5M2 floats with bias and
  saturation
* `vcvtbiasph2hf8` - Convert half-precision floats to E4M3 floats with bias
* `vcvtbiasph2hf8s` - Convert half-precision floats to E4M3 floats with bias and
  saturation

The bias comes in the form of an unsigned 8-bit integer, but it's not
interpreted as such. Conversions to the E5M2 and the M4E3 formats actually treat
it differently.

When converting to the E5M2 format, the 8-bit bias is effectively zero-extended
to be 16-bits wide and then added to the half-precision float's bit-wise
representation.

When converting to the E4M3 format, the 8-bit bias is treated similarly, but the
quantity is shifted right by one place before all that occurs. 

While the intended use-case is surely related to machine learning, I must admit
that I'm unaware of what that is exactly. This difference in how the biases are
treated, and their exact utility is not something that I am able to shed light on.
