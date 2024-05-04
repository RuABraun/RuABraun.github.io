---
layout: post
title: Pytorch Profiler on Wav2vec2 
date: 2022-04-07 12:06:02 +0200
categories: jekyll update
---

WIP

This is going to be an attempted deep dive into using the pytorch profiler for speeding up a model, in this case wav2vec2.

Initial result of putting mask creation into dataloader does not seem to provide significant speedup. There is no noticeable difference in the CPU and CUDA flamegraphs.

The trace shows a gap in GPU usage caused by creating the masks. Just realized that actually because of repeatedly calling "item" the "slow" code was slower than necessary. Actually after changing the code I don't see the gap reduce.

But the gap is quite small.:x

