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
|--------|---|-------|-------|
|        | 0 |   1   | 2     |
|  first | 1 |   0   | 1     |
| second | 2 | 1     | 2     |
| third  | 3 | 2     | 1     |

If word i != word j, going to position costs 1, otherwise 0. "second" is an insertion and which is why at the end the cost in the bottom right is 1. Notice that if all we want to know is WER, you can actually just take that value (1) and divide by the count of reference words (3) to get the WER (33.3%).

However, usually one wants to know how many errors of each type there are, and to do that one needs to get the alignment to then count them. This can be ambiguous, for example: 
