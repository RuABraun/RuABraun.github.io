---
layout: post
title: Why I'm Changing My Mind on E2E ASR 
date: 2021-04-12 12:06:02 +0200
categories: jekyll update
---

WIP

I used to be quite skeptical of E2E ASR. I thought that yes, some elements of the approach were good ideas, but it felt like it was putting too much responsibility on the shoulders of a single system (the neural network) with no priors attached.

First let's clarify what E2E ASR means for me: No phones, the model outputs are letters or subword units. No alignment needed. Using an LM is optional, the model will have already learned an internal LM.

Particularly not using phones bothered knowing how inconsistent pronunciations of English words are, it just seemed suboptimal to have the network be forced to memorize that. 

However, I've had a bit of a change in thinking recently. This has come not from realizing the existance of some highly technical fact, but rather thinking about what the point of doing speech recognition actually is from the perspective of a consumer. There are many different use cases of course, some people just want to take notes without writing, others are transcribing interviews allowing them to easily skim, highlight and summarize it, captioning for lectures or parlament sessions, then there's the large enterprise sector such as call centers, financial compliance and so on.

In all of these what the end-user wants is a clear transcript. It should be easy to read, free of disfluences and repetitions. I believe the traditional approach is flawed for achieving this. 

Traditional ASR is split up into at least two components, the AM and LM. The AM is tasked with modeling the acoustics by recognizing phones. When we then want to output words given some input audio the LM helps choose a sequence of phones that match a sequence of likely words. But because of this split in the modeling it's hard for the model to learn to output clean transcripts, since the better the AM gets at modeling phones the harder it is to ignore certain sounds in the input audio.

Of course, people with some ASR experience will know that in reality traditional ASR does not have a problem with outputting clean transcripts. The LM alone should make sure only reasonable word sequences are output. And in practice the AM learns from the training data which has no disfluencies, so then the AM learns to map those sounds, which really correspond to some phone, to silence (the LM also matters of course).

But the point is that the traditional approach has imperfections. And if our AM is not actually modeling sounds, well then why not just make it model letters? 

While disfluencies are never wanted, it's pretty common to have "you know" missing in a transcript and those words are definitely something the model should recognize correctly. How is a traditional ASR system supposed to deal with these sorts of errors? \\
It cannot. But a transformer-based E2E ASR system can. Thanks to the fact that it looks as much more context (basically seeing the entire input), and that it models letters/subwords directly, it can (for example) learn that a certain sound sequence said quickly at the end of certain sentences ("I really like him yaknow") can be ignored. This is much harder for traditional system to learn this because the AM does not have so much context it can look at, and even if it did the internal representations would have little to do with words - so it doesn't know what sentence is being said - as its task is discriminating phones. 

This makes me wonder whether E2E ASR might be more robust to "imperfect" training transcripts? Note I put imperfect in quotes, based on the motivation of having a clean transcript outputting "I really like him" could be considered the perfect transcript! Regardless, the point is the E2E system is better suited for learning the sort of outputs a user wants to see. 

I plan on putting significantly more time in trying out E2E models. One flaw I still see no solution for is dealing with OOV words, these are often proper nouns and therefore it's unlikely the internal p2g model an E2E model must learn will work. But the possibilities of multilingual models with CTC excite me a lot...
