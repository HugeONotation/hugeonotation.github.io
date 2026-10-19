---
layout: pblog_post
title: SIMD Tokenization
description: An exploration of SIMD-friendly tokenization strategies
---

Lexical tokenization is the process of identifying lexical tokens. In the
abstract it's somewhat analogous to identifying every word and piece of
punctuation in a piece of natural language text. But when it comes to tokenizing
programming languages, this generally means identifying operators, brackets,
identifiers, string literals, comments, etc.

The most common and simple approaches for how to accomplish this generally
feature a loop which iterates over the characters in the input sequence,
consuming characters as tokens are recognized. With different tokens having very
different appearances from one another, it's common for the body of this loop to
feature a large amount of branches checking for various possibilities, at least
in more naive implementations.

Regardless, from what I've seen, most people who try to write a tokenizer fall
into a pattern of utilizing a number of unpredictable branches and a data flow
that's fairly serial, if not outright so. However, as someone with a personal
interest in SIMD vectorization, these things are anathema to my sensibilities.
As such, I've been mulling over the problem of how a tokenizer could be created
in such a way that it's amenable to SIMD vectorization. Partially, this is
motivated by a desired to leverage instruction sets such as AVX-512 on CPUs, but
it's also been motivated by the potential to have the tokenizer run on GPUs.

This post is intended to share some ideas I've been playing with and solutions
to the problems I've identified while trying to construct a parallel-friendly
solution to the tokenization of a formal grammar. For the sake of trying to make
my explanations as accessible as possible, I present my ideas here as a series
of explorations of problems which gradually increase in complexity. This
continues until reaching a point where the language which this tokenizer handles
is suitable for practical use (according to my admittedly subjective judgement). 

Solutions to the problems explored here are provided in a psuedocode that
refrains avoids the fine details of how they would be implemented for a concrete
piece of hardware. The irregular landscape of SIMD instruction sets make it
difficult to get concrete without derailing the conversation because the number
of small details that would need to be addressed is quite high. It is intended
that the primary value of this writeup will be the conceptual approaches
explored. The psuedocode solutions may not be real implementations, but the set
of techniques and primitive operations which are used in these solutions are
chosen to amenable to SIMD vectorized implementations.

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
be implemented in a SIMD-vectorized manner. 

## The Bouncer At The Door 
Let's make the previous problem slightly more realistic by reporting
unrecognized ASCII characters. In a real tokenizer, unrecognized characters are
likely to need special handling. For the immediate future, we'll assume that
unrecognized characters should be reported in the form of an error message, for
which we will need to do nothing more than to .

For such a purpose, it suffices to know the positions will need to know where
the errors occurred so we can properly report their locations.

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

## Memorable Quotes
Suppose that instead of having a language with a single token consisting of the
letter `a`, the language only recognizes quoted strings. In particular, any
double quote character can be considered to be toggling whether you are
currently inside of a quoted string or not. Within a quoted string, we'll make
the assumption that unrecognized characters may be present. The idea is that
within the language we're gradually building up will treat the contents of
string literals as raw bytes so unrecognized characters, or even bytes that
don't map to any ASCII character whatsoever, are not an issue.

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
ends with a long sequence of back slashes: `"...\\\\\\\\\\\\\\\\\\\\"`. The
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

## Character Development
Character literals are generally expressed very similarly to string literals
with the main difference being that single quotes are used instead of double
quotes. Recognizing single quote tokens in the inputs sequence is almost be
identical to identifying double quotes and is naturally not a novel challenge
here. But there is a substantially greater challenge to consider with regards to
identifying the beginnings and openings of these tokens, particularly when you
have a language with both character literals and string literals.

The root of the complexity comes from the fact that single quotes may appear in
string literals and double quotes may appear in character literals. This
requires that we consider whether instances of single and double quotes be taken
into consideration when pairing them off. In a worst case scenario, this
introduces a horizontal data dependency that requires information.

To illustrate this, consider the following example:

| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| " | ' | " |   | ' | " | ' |   | " | ' | " |
{: .table.align_left .table.narrow }

When looking at the `"` located at index 8, it is the start of a string literal.
This is despite the fact that the number of `"`s which appear before it is an
odd number, which previously would have been enough to determine that the `"` at
index 8 was actually closing a string literal. But this is no longer the case,
and we must factor in more information from the preceding context to reach this
conclusion. 
  
To determine that index 8 is the start of a string literal it is necessary to
first determine two pieces of information: first, that you are not currently in
a string literal, as that would mean this double quote ends it, and second, that
you're not currently in a character literal, as the double quote would not
participate in denoting a string literal at all. The former/latter piece of
information requires determining the eveness of preceding double/single quotes
which are not in character/string literals, respectively. You could run a
similar analysis on any other index containing a single or double quote and
reach a similar conclusion with regards to what information is necessary to
determine how they contribute to these tokens, if they do at all.

Hence, to determine that index 8 starts a string literal you have to answer
those two same questions for every single preceding single and double quote
character. This means that the result for every element in the input sequence
there is a chain of data dependencies on potentially any and all previous
elements. This presents a challenge to vectorization as this kinda of dependency
cannot be tackled as straightforwardly as previous problems. That said, it is
still technically possible to create a solution to this problem which is
suitable to SIMD vectorization, although it's admittedly not as elegant as
pervious solutions.

The core idea is to adopt an augmented divide-and-conquer framing of the
problem. We can start with the base case. We can consider each character in the
input sequence as being its own subproblem where our goal is to .

TODO: Complete section

The base case would be to transform each input character as a single block, and
in the recursive step, adjacent pairs of blocks are combined to form new blocks
twice the size. This is repeated until the block size is equal to or greater
than the input sequence size. However, this framing does not factor in that data
dependency on preceding elements. The result of creating a block from a
character and the result of combing two different blocks is also contingent on
information which is propagated from preceding blocks. Hence, there is a need
for augmenting this approach with some concept of propagated information. 

Recall that the goal we're aiming for is to go from an inputs character sequence
to an array where each entry denotes the beginning/end of a string/character
literal, or none of the above. 

When creating size-1 blocks from individual characters, the idea is to translate
the input character sequence .

TODO: Complete

Admittedly, the solution to which I'm referring to is not as elegant as the
previous solutions I've presented. Additionally, I would expect it to perform
notably better on SIMT-based GPU architectures than on CPU architectures which
implement what I'm going to called packed-style SIMD. Additionally, I would
expect that the solution.

TODO: Complete

## Unsolicited Commentary

## Learning Our ABCs
Needless to say, having tokens that literally only consist of `a` is very
unrealistic. Letters are often part of what I'm simply going to refers to as
"textual tokens" in programming languages, a term that deliberately handwaves
some intricacies of real languages by collectively addressing both keywords and
identifiers. These textual tokens typically recognize letters (regardless of
capitalization), underscores, and numeric digits. Not to mention that they are
comprised of more than just one character. However, digits introduce more
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

## Numerical Numbers
In addition to recognizing textual token, a practical language will have a need
to recognize what might be considered to be numeric literals. For out purposes,
we will say that a numeric literal begins with a digit 0-9, and is followed by a
sequence comprised of letters, digits, periods, and underscores. Underscores may
not be recognized as a typical part of how numeric quantities are expressed, but
we'll be using them for the purpose of separators. These are merely meant to be
a convenient means to group the numeric literal's characters for the sake of
visual clarity. C++ uses a single quote `'` for this purpose, but other
languages such as Java and Go choose the underscore `_`. I am personally partial
to the underscore because a simple thin horizontal line is rather difficult to
mistake for any letter or digit.

As a bit of an aside, you may note that no consideration is made for the
potential of a leading `+` or `-`. I will simply handwave this possibility away
by simply saying that in the grammar of this language these would be separate
tokens, unary `+` and unary `-` to bit as directly as possible. This may seem
like a bit of a cop out but this design choice is already widely seen in
mainstream programming languages. The fact that it happens to make our job
slightly easier here is an opportunity that I more than welcome.

There are only two fundamental differences between identifying a textual token
and a numeric token. The first is that the first character must be a digit, and
the second is that a period may be part of a numeric literal.

## Playing Matchmaker


## Pairing Off Brackets

## Going International
so far, it's been assumed that the input sequence consists of ASCII characters,
but this is a simplification of a world that runs on Unicode, and where the
UTF-8 encoding .

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
consideration is made for them in the first place. Yet as this problem of
vectorizing tokenization demonstrates, horizontal data flow can be crucial to
vectorizing certain problems.

The current hardware landscape is currently on a well-established trajectory
where parallelism is the major source of performance gains. This has created .

While technologies such as mutithreading and GPGPU parallelism often get more
attention, there are.
