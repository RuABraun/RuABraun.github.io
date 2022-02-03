---
layout: post
title: Why is the perplexity good but the WER not 
date: 2021-11-21 12:06:02 +0200
categories: jekyll update
---

WIP

This is an age-old question.


Reason 1: Your test and training text is full of filler words like "um" and your LM got better at modeling it, but "um" provides no discrimination about what word comes after so your WER stays the same.
