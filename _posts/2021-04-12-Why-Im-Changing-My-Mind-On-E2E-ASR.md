---
layout: post
title: Changing My Mind On E2E ASR 
date: 2021-04-12 12:06:02 +0200
categories: jekyll update
---

I used to be quite skeptical of E2E ASR. I thought that yes, the approach was interesting and worth investigating, but it felt like it was putting too much responsibility on the shoulders of a single system (the neural network) with no priors attached. It did not feel like there was an advantage to it other than simplicity (which by itself will not help performance).

First let's clarify what E2E ASR means for me: No phones, the model outputs are letters or subword units. No alignment needed and the model learns an internal LM.

Particularly not using phones bothered me knowing how inconsistent pronunciations of English words are, it just seemed suboptimal to force the network to memorize that. 

However, I've had a bit of a change in thinking recently. This has come not from realizing the existence of some technical fact, but rather thinking about what the point of doing speech recognition actually is from the perspective of a consumer. There are many different use cases of course but in by far the majority of them what the end-user wants is a clean transcript. It should be easy to read, free of disfluencies, filler words and repetitions.\\
I believe traditional ASR is flawed for achieving this, and E2E ASR is not. 

Traditional ASR is split up into at least two components, the AM and LM. The AM is tasked with modeling the acoustics by recognizing phones. Training it means tuning the model so that it makes a decision to which phone each frame (25ms of audio) belongs to. But outputting clean transcripts means ignoring parts of the audio. So this design decision of having one model which classifies phones makes it hard for the model to learn to output clean transcripts, since the better the AM gets at modeling phones the harder it is to ignore certain sounds in the input audio.

Of course, people with some ASR experience will know that in reality traditional ASR does not have such a problem with outputting clean transcripts. The LM alone should make sure only reasonable word sequences are output. And in practice the AM learns from the training data which (often) has no disfluencies, so the AM learns to map those sounds, which really correspond to some phone, to silence.

Another issue is that people will regularly pronounce words differently than what you would expect, so during training the model will have to learn to map from one phone to another simply because the lexicon entry for a word will not always correspond exactly how people actually pronounce it. 

So if our AM is not actually modeling sounds, and we're already forcing it do some more complicated memorization, well then why not just make it model letters? 

While fillers like "um" are never wanted, it's pretty common to have "you know" missing in a transcript and those words are definitely something the model should not learn to ignore. How is a traditional ASR system supposed to deal with these sorts of errors? \\
It cannot. But a transformer-based E2E ASR system can. Thanks to the fact that it looks at much more context (basically seeing the entire input), and that it models letters/subwords directly, it can (for example) learn that a certain sound sequence said quickly at the end of certain sentences ("I really like him yaknow") can be ignored. This is much harder for a traditional ASR system to learn because the AM does not have so much context it can look at, and even if it did the internal representations would have little to do with words - so it doesn't know what sentence is being said - as its task is discriminating phones. 


So there's two points I'm making here: (1) With the training data we have we force the AM to learn an ill-defined task (the task is actually more complicated than just learning to classify phones). (2) E2E systems that use lots of context are better suited to outputting clean transcripts because they combine the AM and LM, so they can learn to ignore sounds because of words that were said before and/or after.

Clean transcripts aren't just better for a human reader, it also makes post processing easier for any downstream ML system. And in my experience speech recognition by itself does not have that much value (from a commercial perspective), it's by adding an NLU system on top that a lot more possibilities for use cases open up. 

I feel like some people have spent so much time thinking about how to model phones that they've forgotten that we don't actually care about phones at all. It could be an interesting question for linguists to study, what phones do people use etc., but if your task is speech recognition classifying which sounds were in the audio should only be a means to an end. A more general conclusion is: **Speech recognition is actually not about what a person said, rather it's about what they meant to say.** 

Of course, it's not very satisfying to just cross your fingers and train a NN to do ASR. It would be nice to somehow give the model a good prior. But with wav2vec2 I feel a good solution has been found as the performance is very good! This combined with the above perspective has changed my mind on E2E-ASR: I believe it is the way forward. 

edit: In hindsight one thing I want to clarify, although I keep mentioning transformer models those are not really required. What really matters is just that the model is bidirectional, therefore can see into both the past and future, and of course that the outputs are character based and the model learns an internal LM.
