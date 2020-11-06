---
layout: post
title: Doing non-standard stuff with kaldi decoding
date: 2020-11-06 12:06:02 +0200
categories: jekyll update
---

Here I'm going to describe methods for using kaldi for decoding when you want to do something a bit custom.

## No HCLG just G

Imagine you have an acoustic model that you created via pytorch, and you want to then evaluate how well it does at recognizing phonemes (TIMIT for example). You have a phone LM that you want to incorporate. I'm going to show you how you can do that with kaldi.

Usually kaldi assumes that you have an HMM graph (the H in HCLG) and possibly context dependency as well (the C), then there's also the lexicon etc. If you had a pytorch AM that you had trained on the PDF-ids (the targets you get by calling ali-to-pdf on the alignments), you would then use that by saving the loglikelihood output of your NN to a file, then passing that file and the final.mdl, HCLG.fst to latgen-faster-mapped, which will do decoding and generate a lattice (from which you could then take the best path for example).

But if you want to just model phones, you don't care about the HCL at all. You really just want to use a G that will act as the phone LM.

So the first issue to get around is to avoid doing any mapping from transition IDs (the typical input symbols of the HCLG) to PDF-IDs. We can do that by creating a file latgen-faster.cc with the only difference being we use `DecodableMatrixScaled` instead of `DecodableMatrixScaledMapped` in the code.

Then you take the arpa LM that you have trained, convert it using `arpa2fst`, convert the #0 arcs to <eps> and add self-loops to states that have incoming arcs with input labels not equal 0. The point of that is when the AM predicts a sequence "ae ae ae b b" over five frames, the LM should not care about repetitions, we want it to only score the transition "ae" to "b", and of course we don't want repetitions in the output transcript. Having self-loops fixes that. 

Then you call
```
latgen-faster --acoustic-scale=1.0 --beam=13 --determinize-lattice=false G.fst ark,t:loglikelihoods-file> ark:- | lattice-best-path etc.
```
to get a hypothesis that you can compare to the reference to get the PER!

If you wanted to recognize words, all you'd need to do is create a L.fst, compose that with the G.fst (which is then a word LM) and use the LG.fst.

## G with keyword list

Imagine you just want to recognize a small number of words and you don't want to have any sort of prior regarding which one is more likely.

To do that you just have to create an FST with a single state, and self-loops where each self-loop has a keyword as the input and output symbol.

In pseudo-code
```
fst = Fst()
state = fst.add_state()
fst.set_start(state)
fst.set_final(state)
for keyword in keywords
    fst.add_arc(state, state, keyword, keyword, cost)
fst.write(...)
```

The self-loops should have costs, what these should be you will have to find out experimentally.

You probably will have to create the lang/ folder again. To do that just call `prepare_lang.sh` with the dict/ of keywords and with the `--phone-symbol-table` option set to the `phones.txt` that you trained the AM model with.
