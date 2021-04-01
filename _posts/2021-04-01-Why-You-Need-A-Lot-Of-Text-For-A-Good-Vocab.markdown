---
layout: post
title: Why you need a lot of text to get a good language model 
date: 2021-04-01 12:06:02 +0200
categories: jekyll update
---

WIP

I have a text corpus with nearly 90 million words. Seems enough to get a decent model no? Let's see. The first thing to realise is just covering relatively normal words requires having several hundred thousand words in the vocabulary. Let's what happens when I get a count of all words and see what is at the nth position.
```
199989 krisenbewältigungen 2
199999 gendersensitiv 2
200002 umgehbar 2
200005 widersinnigen 2
200016 ausmehrungen 2
```
Some of these are completely legitimite words. Good we have them in our vocabulary! But (!) the thing to realize is that their counts are very low. We will never be able to actually learn good models for these words because they appear so inoften in our training corpus. Note that the count of 2 starts from the ~160 000th word!<sup>1</sup>

It is kind of expected for this to happen, since it is well known that word counts follow zipf's law, which very simply states that as you go down a table of words sorted by count, their counts decrease very rapidly, meaning a very large amount of the probability mass is covered by the top words. Here is an image

<img src="{{site.url}}/images/words.png" style="display: block; margin: auto;" />

Look at it! (yes I know log scale bla bla, shush pedants) Very quickly the counts drop to nothing. The additional thing to realise is that because the counts are so low there will be a **ton** of noise in the results. Depending on the corpus some words whose "true" probability is higher will be much lower and vice versa. This is bad.

But the main point is that you need to have seen a word a couple times to know how it is used. But because in language there is a looong tail of words that are used infrequently (but are still normal enough that you do want to estimate them!) you need a **lot** of text to get the counts to a reasonable level.

This is why you need to train on billion word corpuses to get good results. Then all those 2s will turn into 20s, and then the model can learn something.

<sup>1</sup> The total count is around 400k by the way (a good chunk of them are rubbish).
