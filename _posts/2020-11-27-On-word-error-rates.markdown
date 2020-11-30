---
layout: post
title: On WER in ASR 
date: 2020-11-06 12:06:02 +0200
categories: jekyll update
---

WIP

This post is going to be a refresher on WER calculation, and then an introduction to a new tool I created which does things a bit differently.

## WER calculation recap

Given a model that outputs a hypothesis the Word-Error-Rate is defined as the number of insertion, deletion and substitution errors the model makes over the count of words in the reference.  

To find out the types of errors one has to align the hypothesis to the reference, this is typically done by creating a cost matrix and backtracing from the bottom right to the start (top left) to find the alignment. Example:

|        |   | first | third |
|:--------:|---:|:-------:|:-------:|
|        | 0 |   1   | 2     |
|  first | 1 |   0   | 1     |
| second | 2 | 1     | 2     |
| third  | 3 | 2     | 1     |

If word i != word j, going to position costs 1, otherwise 0. "second" is an insertion and which is why at the end the cost in the bottom right is 1. Notice that if all we want to know is WER, you can actually just take that value (1) and divide by the count of reference words (3) to get the WER (33.3%).

However, usually one wants to know how many errors of each type there are, and to do that one needs to get the alignment to then count them. This can be ambiguous, for example consider two sentences "first second insertion third" and "first other third". There are different ways to align this that are equally valid:

| first | word | in | sentence |
|:-----:|:------:|:---------:|:-----:|
| first |  ward |     -     | sentence |

and 

| first | word | in | sentence |
|:-----:|:------:|:---------:|:-----:|
| first |  - |     ward     | sentence |

Clearly the pair "word"/"ward" is a more likely substitution error than "in"/"ward", but this alignment method has no way of identifying that.

This has no impact on the WER, as in both cases there are two (one insertion/deletion depending on which sentence is considered the reference, one substitution), but from the perspective of analysing what sorts of errors a model is making - which sorts of words the model is failing to recognize (deletions), which words are confused with each other (substitutions) - having a different alignment will change the result. 

## Getting better alignments

As just mentioned, the traditional method of alignment (which we need to do to get statistics for the different error types) leads to ambiguous alignments with no sensible way of resolving them. `texterrors` is meant to be a tool for getting detailed error metrics which are sensitive to suboptimal alignments. It deals with this issue by, instead of having a cost of 1 for the substitution cost (in the cost matrix), incorporating the character edit distance between the words compared.

Concretely, the substitution cost is set to the edit distance between two words divided by the maximum edit distance possible (length of the longer word), so it is a value between 0 and 1. That way alignments will be favored where words which are similar to each other are substitution errors instead of deletion/insertion errors.\\
Example cost matrix:

|          |   | first | ward | sentence |
|:--------:|:-:|:-----:|:----:|----------|
|          | 0 |   1   |   2  | 3        |
| first    | 1 | 0     | 1    | 2        |
| word     | 2 | 1     | 0.25 | 1.25     |
| in       | 3 | 2     | 1.25 | 1.125    |
| sentence | 4 | 3     | 2.25 | 1.25     |

As one can see here, one incurs less cost by pairing "word"/"ward" instead of "in"/"ward".
To backtrace you find the `cell_x` so that `cell_x` + `transition_cost` = `current_cell`, where `cell_x` is either to the left, diagonal or above the `current_cell`.

This leads to better alignments, and therefore better error statistics. 
