---
layout: post
title: Deriving BPE from scratch
date: 2021-02-09 12:06:02 +0200
categories: jekyll update
---

BPE is a remarkably effective algorithm for finding a set of subwords. Just count pairs of tokens, merge the most frequent one, repeat until you have the desired number of subwords. Why does this work, and why would just picking the k most frequent ngrams not?

Let's create a simple 'corpus'\* (word counts are what actually matter):
```
low 5
lowest 2
newer 6
wider 3
new 2
```

Including the separator (\_, appears implicitly at the end of a word) there are 11 characters. Let's pick the 20 most common ngrams, they turn out to be (in order, but starting with unigrams):
```
['l', 'o', 'w', '_', 'e', 's', 't', 'n', 'r', 'i', 'd', 'er', 'er_', 'r_', 'we', 'ne', 'new', 'ew', 'lo', 'low']
```

This is what BPE outputs (single character tokens are not ordered):

```
['l', 'o', 'w', '_', 'e', 's', 't', 'n', 'r', 'i', 'd', 'er', 'er_', 'ne', 'new', 'lo', 'low', 'newer_', 'low_', 'wi']
```

The first not-unigram is the same for both ("er"), the second as well, but then the methods diverge. Note how the ngram counter picks up "we" while BPE never does. Why is that? Normally "we" would of course be a reasonable token, but for this corpus it makes more sense intuitively, considering the counts, to keep it seperate so that low + est and new + er are valid tokenisations.

Because BPE redoes the count each time two tokens are merged (from which a new one is created), and only the merged token is counted - instead of also the previous two - the counts of some tokens can go down after a recount. So when "er" is created the count of "we" drops from 8 to 2! The recounting makes sense in hindsight, the counts of a token should be based on when that token could actually be used, and after choosing the "er" token, the token "we" is not possible as often as before.

What if we change the k most frequent method to iteratively merge with recounts inbetween, and when recounting if two tokens were merged then we only count the merged token (not the previously split tokens)?

We would have to start with some set of tokens, those could be the unigrams. Then we count ngrams that are not already in our token set, merge the most frequent, repeat. This is basically BPE, except we're still doing a bunch of wasted computation by counting ngrams: It should be, with a little thinking, obvious that a bigram will always be at least as likely as a trigram. So rather than count all ngrams that are not already in the token set, we just count bigrams of tokens AKA pairs of tokens. This is BPE.

Now the motivation for the algorithm of BPE is clear. It recounts after each merge to make sure the statistics used are up-to-date, and by considering pairs of tokens it saves a lot of computation.


\* Taken from Jurafsky's chapter on BPE.
