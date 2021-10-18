---
layout: post
title: Notes on Decoupled Contrastive Learning 
date: 2021-10-15 12:06:02 +0200
categories: jekyll update
---

Trying to summarize the recent paper [Decoupled Contrastive Learning](https://arxiv.org/abs/2110.06848).

The contrastive loss is defined as 
$$L_i=-\log \frac{e^{z_i^{(1)} \cdot z_i^{(2)} / \tau }}{\sum_j e^{z_i^{(1)} \cdot z_j^{(2)} / \tau}}$$

for one positive pair $z_i^{(1)}, z_i^{(2)}$. Minimizing the loss means the numerator needs to be increased (by increasing the dot product of the positive pair) while decreasing the denominator (decreasing dot product of negative pairs). As the loss goes down the ratio should go to 1.

Through some rearranging they find that there's a shared term in the gradients of $z_i^{(1)}, z_i^{(2)}, z_j$:

$$q_B^{1} = 1 - \frac{e^{z_i^{(1)} \cdot z_i^{(2)} / \tau }}{\sum_j e^{z_i^{(1)} \cdot z_j^{(2)} / \tau}} $$

The fraction is the same as in the loss. So when the ratio is close to 1 the gradient is small. The end result is that if there are no difficult negatives then the gradient will be small. Even if the positive pair is not that close to each other!
This slows learning down and is why large batch sizes were required to ensure the existence of hard negatives.

They empirically show that when using small batch sizes the distribution of $q_B$ has a strong bias towards small values (<<0.5). This changes as the batch is increased.

They propose removing the positive pair from the denominator:
$$L_i^{DCL}=-\log \frac{e^{z_i^{(1)} \cdot z_i^{(2)} / \tau }}{\sum_{j \ne i} e^{z_i^{(1)} \cdot z_j^{(2)} / \tau}}$$

And a weighting function to ensure that the gradient is larger when positive samples are further from each other.

Experiments show learning is faster.

Personal thoughts: I had wondered whether it might be suboptimal to include the positive pair in the denominator! Interesting to see this feeling validated! I hesitated pursuing this because allowing the ratio inside the log to be larger than 1 seemed wrong.

