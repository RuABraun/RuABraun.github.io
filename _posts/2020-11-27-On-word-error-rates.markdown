---
layout: post
title: On WER in ASR 
date: 2020-11-27 12:06:02 +0200
categories: jekyll update
---

This post is split in two parts: First a refresher on standard WER calculation and an illustration of how this can be suboptimal when interested in analysing errors. Then an introduction to a different approach used by my [new tool](https://github.com/RuABraun/texterrors) which fixes the problems and gets better error metrics. You can skip to the second part by clicking [here](#newtool).

## WER calculation recap

Given a model that outputs a hypothesis the Word-Error-Rate is defined as the number of insertion, deletion and substitution errors the model makes over the count of words in the reference.  

To find out the types of errors one has to align the hypothesis to the reference, this is typically done by creating a cost matrix and backtracing from the end (bottom right) to the start (top left) to find the alignment. Example:

|        |   | first | third |
|:--------:|---:|:-------:|:-------:|
|        | 0 |   1   | 2     |
|  first | 1 |   0   | 1     |
| second | 2 | 1     | 1     |
| third  | 3 | 2     | 1     |

Taking a horizontal or vertical (deletion/insertion) transition costs 1. Taking a diagonal to position `i, j` costs 1 if word i != word j, else 0. "second" is not recognized by the model (a deletion error) which is why at the end the cost in the bottom right is 1. Notice that if all we want to know is WER, you can actually just take that value (1) and divide by the count of reference words (3) to get the WER (33.3%).\\

However, usually one wants to know how many errors of each type there are, and to do that one needs to get the alignment to then count them. This requires backtracing which is done by finding the `cell_x` so that `cell_x` + `transition_cost` = `current_cell`, where `cell_x` is either to the left, diagonal or above the `current_cell` (repeat process until the start is reached).\\
This can be ambiguous, for example consider two sentences "first word in sentence" and "first ward sentence". There are different ways to align this:

| first | word | in | sentence |
|:-----:|:------:|:---------:|:-----:|
| first |  ward |     -     | sentence |

and 

| first | word | in | sentence |
|:-----:|:------:|:---------:|:-----:|
| first |  - |     ward     | sentence |

Clearly the pair "word"/"ward" is a more likely substitution error than "in"/"ward", but this alignment method has no way of identifying that since the cumulative costs are the same, see cost matrix:

|          |   | first | ward | sentence |
|:--------:|:-:|:-----:|:----:|:--------:|
|          | 0 |   1   |   2  |     3    |
|   first  | 1 |   0   |   1  |     2    |
|   word   | 2 |   1   |   1  |     2    |
|    in    | 3 |   2   |   2  |     2    |
| sentence | 4 |   3   |   3  |     2    |

This has no impact on the WER, as in both cases there are two (one insertion/deletion depending on which sentence is considered the reference, one substitution), but from the perspective of analysing what sorts of errors a model is making - which sorts of words the model is failing to recognize (deletions), which words are confused with each other (substitutions) - having a different alignment will change the result. 

## Getting better alignments with "texterrors" <a name="newtool"></a>

As just mentioned, the traditional method of alignment (which we need to do to get statistics for the different error types) leads to ambiguous alignments with no sensible way of resolving them. ["texterrors"](https://github.com/RuABraun/texterrors) is meant to be a tool for getting detailed error metrics. As these are sensitive to suboptimal alignments it uses a smarter method: Instead of having a cost of 1 for the substitution cost (in the cost matrix), it incorporates the character edit distance between the words compared.

Concretely, the substitution cost is set to the edit distance between two words divided by the maximum edit distance possible (length of the longer word), so it is a value between 0 and 1 (slightly more complicated in practice, you'll see later why). That way alignments will be favored when words which are similar to each other are substitution errors instead of deletion/insertion errors.\\

Example cost matrix:

|          |   | first | ward | sentence |
|:---:|:-:|:---:|:---:|---|
|          | 0 |   1   |   2  | 3        |
| first    | 1 | 0     | 1    | 2        |
| word     | 2 | 1     | 0.25 | 1.25     |
| in       | 3 | 2     | 1.25 | 1.125    |
| sentence | 4 | 3     | 2.25 | 1.25     |

As one can see here, the best (lowest) cumulative cost is achieved by pairing "word"/"ward".

There is still one last issue to deal with, see this example:

|           |    | hello | speedbird |  six  |  two  |
|:---:|:--:|:---:|:---:|:---:|:---:|
|           |  0 |   1   |     2     |   3   |   4.  |
| speedbird | 1. | 0.89 |     1     |   2   |   3   |
|   eight   | 2. | 1.89 |   1.78   |  1.8  |  2.8  |
|    six    | 3. | 2.89 |   2.67   | 1.78 | 2.78 |
|    two    |  4 |  3.8  |   3.67   | 2.78 | 1.78 |

The alignment will end up being

| speedbird | eight | six | two |
|:-----:|:------:|:---------:|:-----:|
| hello | speedbird | six | two |

This happens here because using the edit distance leads to the substitution cost often being smaller than the insertion/deletion cost and therefore alignments with more substitutions are favored.\\
This sort of bad alignment also happens with normal costs of 1/1/1 for ins/del/sub (consider the above example, as the costs are the same for different errors it depends on the implementation which alignment is chosen, you can think of it as random). That's why such tools, when meant to be used for getting detailed error metrics, will increase the substitution cost to improve alignments. We can do the same: `texterrors` will after the previously mentioned calculation times the cost by 1.5. This will lead to the following cost matrix:


|           |    | hello | speedbird | six | two |
|:---------:|:--:|:-----:|:---------:|:---:|:---:|
|           |  0 |   1   |     2     |  3  |  4. |
| speedbird | 1. |  1.3  |     1     |  2  |  3. |
|   eight   | 2. |  2.3  |     2.    | 2.2 | 3.2 |
|    six    | 3. |  3.3  |     3.    |  2. |  3. |
|    two    |  4 |  4.2  |     4.    |  3. |  2. |

And the following (obviously superior) alignment.

| - | speedbird | eight | six | two |
|:-----:|:------:|:---------:|:-----:|:---:|
| hello | speedbird | - | six | two |

This leads to better alignments, and therefore better error statistics. 

Of course, if one has time stamps one automatically has an alignment and doesn't need to worry about any of this. I'm going to add support for `ctm` files soon so if one wants to one can use them instead.
