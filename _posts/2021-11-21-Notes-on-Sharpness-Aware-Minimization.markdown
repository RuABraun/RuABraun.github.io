---
layout: post
title: Notes on Sharpness Aware Minimization and LM paper using it
date: 2021-11-21 12:06:02 +0200
categories: jekyll update
---

WIP

Notes on [SAM paper](https://arxiv.org/pdf/2010.01412.pdf) and the child paper using [SAM for language modeling](https://arxiv.org/pdf/2110.08529.pdf).

Basic idea of SAM is to perturb the weights in different directions and minimize the loss where it's at its max.

So instead of just $$\underset{w}{\min} L_S(w)$$ do $$ \underset{w}{\min} \underset{\Vert \epsilon \Vert_{p} \le \rho}{\max} L_S(w + \epsilon)$$, where p equals 2 and $$\rho$$ is a hyperparameter set to a value like 0.05.

They also have an additional term which in practice they set to the L2 loss of the weights. Not clear to me why the term is necessary.

Actually trying out different $$\epsilon$$s would be very costly obviously so instead to do the minimization they first use a taylor expansion like:

$$ \underset{\Vert \epsilon \Vert_p}{\arg\max} L_S(w + \epsilon) \approx \underset{\Vert \epsilon \Vert_p}{\arg\max} L_S(w) + \epsilon \nabla_w L_S(w) = \underset{\Vert \epsilon \Vert_p}{\arg\max} \epsilon \nabla_w L_S(w)$$

The value of $$\epsilon$$ that solves this approximation is "given by the solution to a classical dual norm problem [...]" $$\hat \epsilon$$. This leads to the following expression and loss (from the paper): 

<img src="{{site.url}}/images/sam_lossequation.png" style="display: block; margin: auto;" />

So in the end they end up throwing away the complicated terms and just using the gradient information to climb in the steepest direction. The amount they climb is determined by the hyperparameter $$\rho$$.

Note that this is similar to when trying to find an adversial example: Changing the variables (this time the weights, not the input) by a small amount to maximize the loss. 

The procedure is to compute the loss for weights $$w$$, then use that loss and gradient to compute $$\hat \epsilon$$, perturb the weights, compute the loss again and use that gradient to update. 

There's a pytorch implementation [here](https://github.com/davda54/sam). Code looks quite simple. The result of the first forward-backward is to set `e_w=p.grad*scale` and modify the parameters $$p$$ by `p.add_(e_w)`, upon which after the second forward-backward the parameters are reset and one does a normal update `self.base_optimizer.step()` with the new gradient. 


Looking at it practically, this means the gradients will always comes from a nearby "higher point" (higher as in the loss is higher). What effect will this have on the optimization trajectory? If you're already in a minimum it's already too late I guess. You're just going to be redirected to the one you're already in right? But if you're bouncing between minima then if you land in a sharp one you should get a bigger gradient correction than if you land in a flat minima where the higher points are actually not that high. So I guess it improves the chances of landing in a good minima.  

There's a bunch of other interesting content. One point they mention is that the updates are of course done per batch, which means values like $$\hat \epsilon$$ are only estimated with small number of datapoints rather than the entire dataset. They look into what the effect of using subsets (of the batch) of size $$m$$ which are then used to compute the SAM update across different GPUs and then averaging, and find that having smaller sizes performs better!

The child paper mentions that one can use a fourth smaller batch size in the first step to speed things up with losing performance.

They use $$m$$ to mean the number of subsets that are created from the batch (rather than being $$m$$ being the size of the subset).

They that SAM improves results for all model sizes and especially when the training data is small. The $$\rho$$ does need to be tuned, however they identify 0.15 as a good default. 
