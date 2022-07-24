---
layout: post
title: A comparison of fairseq, speechbrain, k2 for ASR
date: 2022-06-26 12:06:02 +0200
categories: jekyll update
---

WIP

I have had the opportunity to work with some of the different E2E ASR toolkits framework out there and thought it could be useful to give an overview and comparison of these.

This will for each give a high level overview of the code base, what the training loop looks like, and name stuff I liked or disliked. In the end I will make some comments comparing them.
Whenever I refer to files, I assume you are starting from the root folder of the repo.
# fairseq
[Fairseq](https://github.com/facebookresearch/fairseq) is not just meant for ASR. It's for many different modeling tasks, including language modeling, translation, masked LM pretraining, summarization an
d text to speech.

Training models and doing inference is done via command-line scripts (found in `fairseq_cli/`). The training loop and data processing code is unified when possible. As a consequence the code is generally a bit more abstract, you cannot look at just one file if you want to read and understand the training loop.\\
The codebase can be seen as split into four different parts.\\
- `fairseq_cli/` contains CLI scripts for data processing, model training and evaluation.\\
- `fairseq/` (except for `/tasks/`) is like a library of common models, data processing or utility classes and functions\\
- `fairseq/tasks/` has files which contain code specific to a modeling task\\
- `examples/` has the documentation and scripts for training and evaluating models related to a facebook paper(s) and corpus\\

When training you pass a "task" argument (when calling the CLI scripts) which ensures the appropriate code is run for whatever task you are doing. Many modifications to a task, like using a different model, can done by changing a command-line argument or using a config file to redefine a default (the config is based on hydra/omegaconf). Models defined in fairseq (`fairseq/models/`) will have a decorator enabling you to specify it using one keyword (making it easy to change).

There are several tasks related to speech recognition: `speech_to_text`, `speech_recognition`, `audio_pretraining` and `audio_finetuning`. The latter two are for wav2vec[2]. It seems they are planning to unify all the previously mentioned ASR related ones to `speech_to_text`, but this is still WIP.

The data format depends on the task and tends to favor simplicity. For example for wav2vec pretraining you just need a .tsv file which has on the first line a root path, and every line afterwards has a path to an audio file and its sample count. That's it. In `examples/*` there always is an explanation for how to get to the data format needed.

I find fairseq to be a good toolkit for reproducing results from facebook's papers - I was surprised how easy it was to do the pretraining for wav2vec models - and convenient if you want to toy around with a research idea that is slightly different from what already exists. A lot of small changes that you might want to do can be done fairly easily. If you're doing something significantly different from what already exists that could get a lot harder and you will likely have to end up learning about how the entire fairseq repo ties together to achieve that. But I do think once you get it it could be quite nice to work with. One downside is you can't rely much on support from the maintainers; github issues are responded to rarely. It also is not very suited for using in production (but to be fair that is not what it's made for).

Let's skim over the training loop, we start in [fairseq_cli/train.py](https://github.com/facebookresearch/fairseq/blob/main/fairseq_cli/train.py#L180):
```
while epoch_itr.next_epoch_idx <= max_epoch:
    [..]
    valid_losses, should_stop = train(cfg, trainer, task, epoch_itr)
```
With train defined in the same file and then calling [trainer.train_step()](https://github.com/facebookresearch/fairseq/blob/main/fairseq_cli/train.py#L312):

```
logger.info("Start iterating over samples")
for i, samples in enumerate(progress):
     [..]
	 log_output = trainer.train_step(samples)
```
The dataloader is wrapped inside of `progress`.  The trainer object is defined in [fairseq/trainer.py](https://github.com/facebookresearch/fairseq/blob/main/fairseq/trainer.py) and its [train_step()](https://github.com/facebookresearch/fairseq/blob/main/fairseq/trainer.py#L824) calls
```
for i, sample in enumerate(samples):  # delayed update loop
    [..]
	loss, sample_size_i, logging_output = self.task.train_step(
		sample=sample,
		model=self.model,
		criterion=self.criterion,
		optimizer=self.optimizer,
		update_num=self.get_num_updates(),
		ignore_grad=is_dummy_batch,
		**extra_kwargs,
	)
```
the task specific train step. Note that `samples` will have a length bigger 1 only if gradient accumulation is used.

A lot of tasks actually don't have a `train_step()` defined and just use the generic one defined in [fairseq/tasks/fairseq_task.py](https://github.com/facebookresearch/fairseq/blob/main/fairseq/tasks/fairseq_task.py#L490).

# speechbrain

[Speechbrain](https://github.com/speechbrain/speechbrain) is for speech processing tasks, they have an impressive array of examples where they get good results (`recipes/`).  The core training loop is unified in a `Brain` class (found in `speechbrain/core.py`), the data processing code is specified in each recipe.

It uses a powerful yaml config ([HyperPyYaml](https://github.com/speechbrain/HyperPyYAML)) which is generally used not only to define hyperparameters but also to define data processing related things like the data loading sampler, the loss, the optimizer and the model. A complicated model would be defined by refering to the speechbrain implementation (for example a transformer), a simple one like an 3-layer MLP you could define right in the config.

The `Brain` calls is initialized with this config, and will therefore hold a reference to everything related to training such as the model, the loss, the optimizer and so on.

How the model is used is specified by overriding two methods  [of the `Brain` class] `compute_forward` and `compute_objectives`. The former defines how to get from the data to the outputs you need to compute the loss. You then use the latter to define how to compute the loss from those outputs (it's  called automatically after the former). Sometimes the implementation will be extremely simple: In `compute_forward` you call the model and in `compute_objectives` you call the loss defined in your config. If you have a model with separate stages and/or have multiple losses then the methods would be the location to specify how they are connected.

The data is typically stored in CSV format (with filepaths pointing to audio data of course), in your recipe you then specify (via a custom "pipeline" construct) how to go from the data in the CSV to the data you want available in a batch.

I like the config and how powerful it is.  I think they've made some good decisions about how to allow people to make the changes they need to write code specific to their problem. I have though sometimes been bitten by behaviour that I did not expect. It definitely feels more like a framework than a toolkit, something that expects you to do things in a certain way and if you don't you could end up having a hard time. The code is very object oriented.

# k2/icefall/lhotse

This is by far the newest of the three tools I'm comparing here. I wrote k2/icefall/lhotse because there are three different repositories you would make use of: [k2](https://github.com/k2-fsa/k2) for ragged tensors and lattice related operations, [lhotse](https://github.com/lhotse-speech/lhotse) for everything related to data and [icefall](https://github.com/k2-fsa/icefall) which contains recipes using the former two.

So for example icefall contains recipes for training transducer models. The downloading of a corpus, the feature extraction, the code for taking care of dynamic batching will come from lhotse. Ragged tensors, RNNT loss computations, (WFST) decoding graphs will come from k2. Icefall has the contain putting everything together. There is no unified training loop, each recipe will have its own straight-forward implementation making use of lhotse and k2 to do training and inference.

This makes reading the implementation and making changes very easy. I have achieved very good performance and plan to investigate the implementation details. Installing `lhotse` gives access to a very nice CLI that easily allows one to convert from a kaldi style data dir to a lhotse style manifest (actually consisting of two files, one for the audio data and one for the labels) and to extract features. I will say though the decorator ratio in the lhotse codebase is the highest I've ever seen, debugging can be a pain. Which brings me to the fact that this is all very new and in a state of flux, so there are a relatively high amount of bugs. On the bright side everyone involved is very responsive and if something doesn't work one can get help very quickly.
