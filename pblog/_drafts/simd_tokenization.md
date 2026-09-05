---
layout: pblog_post
title: SIMD Tokenization
description: An exploration of SIMD-friendly tokenization strategies
---

Lexical tokenization is the process of identifying lexical tokens. In the
abstract it's somewhat analogous to finding the bounds of every word and piece
of punctuation in a piece of natural language text. But when it comes to
tokenizing programming languages, this generally means identifying operators,
brackets, identifiers, string literals, comments, etc.

The most common and simple approaches for how to accomplish this generally
feature a loop which iterates over the characters in the input sequence,
consuming characters as tokens are recognized. With different tokens having very
different appearances from one another, it's common for the body of this loop to
feature a large amount of branches checking for various possibilities, at least
in more naive implementations.

Regardless, from what I've seen, most people who try to write a tokenizer fall
into a pattern of utilizing a number unpredictable branches and a data flow
that's fairly serial, if not outright so. However, as someone with a personal
interest in SIMD vectorization, these things are anathema to my sensibilities.
As such, I've been considering the problem of how a tokenizer could be created
in such a way that it's amenable to SIMD vectorization. Partially, this is
motivated by a desired to leverage instruction sets such as AVX-512 on CPUs, but
it's been motivated by the potential to have the tokenizer run on GPUs as well.

This post is intended to share some ideas I've played with and solutions to the
problems I've identified while trying to construct a parallel-friendly solution
to the tokenization of a formal grammar. For the sake of trying to make my
explanations as accessible as possible, I present my ideas here as a series of
explorations of problems which gradually increase in complexity.

## Oops! All A's
Suppose a language with a single token, that token being `a`.

Let's assume that our input is a sequence of `N` ASCII characters. Since there
is only a single token, merely reporting the positions at which this token is
found is enough to complete the tokenization.

This problem is easily solved in a vectorized fashion by running the following
logic (expressed in psuedocode) for each byte in the input sequence:

```
Char[N] str = ...
Boolean[N] token_positions;

vectorized for i in 0..N:
    token_positions[i] = (str[i] == 'a');

```

Here I used `vectorized for` as a means to communicate that the loop is meant to
be implemented in a SIMD-vectorized manner. I refrain from being specific with
regards to how things are implemented on a concrete piece of hardware because
the irregular landscape of SIMD instruction set make it difficult to do so in a
way which will be universally applicable. The primary value here is the
conceptual approaches which aim to be applicable to a wide range of machine
architectures.

## The Bouncer At The Door 
Let's make the previous problem slightly more realistic by more properly
addressing the case of unrecognized ASCII characters. In a real tokenizer,
unrecognized characters are likely to need special handling. For the immediate
future, we'll assume that unrecognized characters should be reported in the form
of an error message. For such a purpose, we will need to know where the errors
occurred so we can properly report their locations.

An additional Boolean array can be used to track the positions at which
unrecognized characters were found. (With only `a`s being allowed so far, this
is just the complement of the already existing `token_positions` array, but this
design will flow better into the next scenario.) Once the tokenization pass is
run, error messages may be generated based purely on this aforementioned error
location Boolean array.

```
Char[N] str = ...
Boolean[N] token_positions;
Boolean[N] error_positions;

vectorized for i in 0..N:
    token_positions[i] = (str[i] == 'a');
    error_positions[i] = (str[i] != 'a');

for i in 0..N:
    if (!error_positions[i]) {
        continue;
    }

    String char_hex_string = to_hex_string(to_int(str[i]));
    String pos_string = to_string(i);
    print(
        "Unrecognized character 0x" +
        char_hex_string+
        " found at position "+
        pos_string
    );
```

Here, I've left the error printing unvectorized because it's not a core issue
that I'm interested in tackling here. There are approaches to this problem which
are SIMD-friendly, and perhaps it's worth exploring in the future.

## Space to Breathe
Another step towards a more realistic language is to have the language ignore
whitespaces. This includes spaces, tabs, vertical tabs, newlines, and carriage
returns. While they do not contribute to any recognized token, their presence
also does not trigger any error.

Identifying these characters within the character stream is fairly easy because
many of the aforementioned whitespace characters are encoded using sequential
values:

| Decimal value | Character       |
|---------------|-----------------|
| 9             | HORIZONTAL TAB  |
| 10            | NEW LINE        |
| 11            | VERTICAL TAB    |
| 12            | NEW PAGE        |
| 13            | CARRIAGE RETURN |
| 32            | SPACE           |
{: .table.align_left  }

Hence the test `'\t' <= c && c <= '\r'` covers all but one of the whitespace
characters, and an additional `c == ' '` covers the remaining possibility.

Augmenting our previous approach:

```
Char[N] str = ...
Boolean[N] positions;
Boolean[N] error_positions;

vectorized for i in 0..N:
    Boolean is_whitespace = '(\t' <= str[i] && str[i] <= '\r') || (str[i] == ' ')
    positions[i] = (str[i] == 'a');
    error_positions[i] = (str[i] != 'a') && !is_whitespace;

for i in 0..N:
    if (!error_positions[i]) {
        continue;
    }

    String char_hex_string = to_hex_string(to_int(str[i]));
    String pos_string = to_string(i);
    print(
        "Unrecognized character 0x" +
        char_hex_string+
        " found at position "+
        pos_string
    );
```

And just like that, whitespace is handled appropriately.

## Learning Our ABCs
Needless to say, having tokens that literally only consist of `a` is
unrealistic. Letters are often part of what I'm simply going to refers to as
"textual tokens" in programming languages, a term that deliberately handwaves
some intricacies of real languages by collectively addresses both keywords and
identifiers. These textual tokens typically recognize letters, regardless of
their capitalization, underscores, and numeric digits. Not to mention that they
are comprised of more than just one character. However, digits introduce more
complexity since they also contribute to numeric literals, and variable-length
tokens are still another major leap away.

This can be addressed through a similar generalization on the tests for `a`. It
would be more appropriate to have the program recognize all lowercase and
uppercase letters, and underscores while we're at it since these are generally
the characters which can be used to begin an identifier. Hence, this set of
characters has an eminently practical relevance.

Updating the aforementioned solution to test for these additional cases isn't
quite that difficult.

## Choosing Your Words Carefully
At this point, it's probably a good time to start considering the possibility of
variable-length tokens. In practical programming languages identifiers generally
consist of a sequence of letters, underscores, and digits, but crucially, not
starting with digits.

## Memorable Quotes
Suppose that instead of having a language with a single token consisting of the
letter `a` consists of quoted strings. In particular, any double quote character
can be considered to be toggling whether you are currently inside of a quoted
string or not. Within a quoted string, we'll make the assumption that
unrecognized characters may be present. The idea is that within the language
we're gradually building up will treat the contents of string literals as raw
bytes so unrecognized characters, or even bytes that don't map to any ASCII
character whatsoever, are not an issue.

The first step to tokenizing quoted strings is unsurprisingly identifying double
quotes within the input token sequence, a task which is trivially completed via
vectorized comparisons against '"'. The next task that follows is identifying
the starts of string literals, something which necessarily requires a flow of
information across the width of the input domain. In particular, the parity of
the count of preceding double quotes determines whether some arbitrary double
quote opens or closes a quoted string. An even/odd number of preceding double
quotes indicates that the current double quote opens/closes a quoted string
respectively. However, this dependence on information from preceding positions
in the input sequence doesn't map as well to vectorized processing as the
examples that have been explored here so far. 

It is at this point that we can begin to explore the relevance of prefix scans. 

## The Great Escape
In the previous section, the problem of escape sequences was outright ignored,
but they are a necessary component of any practical tokenizer. They also present
a rather particular challenge to tokenization. Consider a string literal that
ends with a long sequence of back slashes: `"...\\\\\\\\\\\\\\\\\\\\\\\"`. The
question that must be addressed is whether the final `\"` is an escape sequence
for a quote, or whether the last backslash actually belongs to an escape
sequence for a backslash, e.g. `\\`. With a bit of thought, I think it's obvious
that whether or not this is the case is dependent on whether the chain of
backslashes that end the string literal contains an even or an odd number of
backslashes. An even number means all the backslashes are escape sequences for
backslashes, and an odd number means that the final unpaired backslash is part
of an escape sequence for a double quote. Hence, the parity of the backslash
sequence's length is what we're really concerned about.

Addressing this problem can reasonably be done through an algorithm that I
personally call a streak, which is actually a special case of something known as
a segmented scan.

```
T[N] input;
int[N] output;

int count = 1;
for i in 1..N:
    output = count;

    if input[i] == input[i - 1] {
        count += 1;
    } else {
        count = 0
    }
```

What this algorithm accomplishes is to compute the number of preceding elements
whose values are equal to the element at the current array position. This helps
us address the aforementioned problem of knowing whether the previous .

## 

## Playing Matchmaker


## Pairing Off Brackets

## Conclusion
Since around six years or so, I've developed a growing interest in the SIMD
capabilities of modern CPUs. It's an area in which great investment have been
made over the past three decades. Today, SIMD instructions form a strong
majority of the x86 instruction set by just about every metric, and they
comprise a decent portion of just about every other mainstream ISA. Yet despite
this, modern programming languages generally do little if anything to address
this segment of hardware functionality. Some languages do provide types meant to
represent a SIMD vector's worth of data, and associated operators/functions. But
this model of abstracting SIMD treats horizontal operations (those where data
flows across lanes rather than within lanes) as a minor concern, assuming any
consideration is made for them in the first place. 
