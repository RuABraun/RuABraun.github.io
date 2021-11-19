---
layout: post
title: Why you need a billion words to get a good language model 
date: 2021-04-01 12:06:02 +0200
categories: jekyll update
---

I have a German text corpus with nearly 90 million words. Seems enough to create a decent language model no? Let's see. The first thing to realise is just covering relatively normal words requires having several hundred thousand words in the vocabulary. Let's see what happens when I get a count of all words and check what is at the nth position.
```
199989 krisenbewältigungen 2
199999 gendersensitiv 2
200002 umgehbar 2
200005 widersinnigen 2
200016 ausmehrungen 2
```
The words I'm showing here are legitimite. Good we have them in our vocabulary. But (!) their counts are very low. The thing to realize is we will never be able to actually learn good models for these words because they appear so inoften in our training corpus. Note that the count of 2 starts from the ~160 000th word!<sup>1</sup>

It is kind of expected for this to happen, since it is well known that word counts follow zipf's law, which put simply states that as you go down a table of words sorted by count, their counts decrease very rapidly, meaning a very large amount of the probability mass is covered by the top words. Here is an image (forgive the lack of axis labels please, y-axis is count in millions x-axis rank of word): 

<img src="{{site.url}}/images/words.png" style="display: block; margin: auto;" />

Look at it! (yes I know log scale bla bla, shush pedants) The counts drop extremely quickly to almost nothing. 

So the main point is that you need to have seen a word a couple times to know how it is used. But because in language there is a looong tail of words that are used infrequently (but are still normal enough that you do want to estimate them!) you need a **lot** of text to get the counts to a reasonable level.

The additional thing to realise is that because the counts are so low there will be a **ton** of noise in the results. Depending on the corpus some words whose "true" probability is higher will be much lower and vice versa. This is bad.

This is why you need to train on billion word corpuses to get good results. Then all those 2s will turn into 20s, and then the model can learn something.

<sup>1</sup> The total count is around 400k by the way (a good chunk of them are rubbish).
