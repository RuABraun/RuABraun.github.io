---
layout: post
title: Why the Temperature Matters for Contrastive Loss 
---

Contrastive learning has become very popular recently, see [here](https://github.com/HobbitLong/PyContrast/blob/master/AWESOME_CONTRASTIVE_LEARNING.md) for a good overview of $$L$$ recent papers.

However, one thing which they all use but is not well motivated is the use of a temperature parameter in the softmax that the contrastive loss uses. I want to share why I think it matters.

As a reminder this is what the equation looks like:

$$L=-\log \frac{e^{x_p \cdot x_+ \over \tau }}{\sum_i e^{x_p \cdot x_i \over \tau}}$$

Where the numerator contains the comparison between the positive pair $$x_p$$ and $$x_+$$, and the denominator has all the comparisons (all negative pairs except one). Minimizing the loss means maximizing the value in the numerator (maximizing $$x_p \cdot x_+$$) and minimizing the denominator (minimizing all $$x_p \cdot x_i$$).

Now imagine $$\tau=1$$. Remember the similarity function is the cosine distance, which with normalized vectors goes from -1 to 1. After applying $$exp$$ the range goes from 1/e to e.

Note how small that range is, and note that if the vectors were orthogonal to each other than the similarity is 0 and then $$e^{0}=1$$.

So making negative pairs orthogonal to each other is not enough to minimize the loss, because the numerator can at most be $$e$$ while the denominator would be (if all pairs were orthogonal) $$\sum_i 1$$. What the model would have to try and do is make the negative pairs all be antiparallel to each other. That's obviously not ideal, if two vectors are orthogonal that should be enough of a separation.

Now set the $$\tau=0.1$$. Now the range goes from $$e^{-0.1}=4e-5$$ to $$e^{10}=22000$$ ! Incase of orthogonality the similarity after exp is still 1 of course. But now making the vectors of a negative pair orthogonal (rather than close to parallel) will result in a much larger decrease in loss, as the numerator value can be so much larger (i.e. $$22000 >> \sum_i 1$$).
