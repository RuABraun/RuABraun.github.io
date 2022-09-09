---
layout: post
title: A comparison of fairseq, speechbrain, k2 for ASR
date: 2022-06-26 12:06:02 +0200
categories: jekyll update
---

I have had the opportunity to work with some of the different E2E ASR toolkits framework out there and thought it could be useful to give an overview and comparison of these.

This will give for each a high level overview of the code base, what the training loop looks like, and name stuff I liked or disliked.
Whenever I refer to files, I assume you are starting from the root folder of the repo.

# fairseq
[Fairseq](https://github.com/facebookresearch/fairseq) is not just meant for ASR. It's for many different modeling tasks, including language modeling, translation, masked LM pretraining, summarization an
d text to speech.

Training models and doing inference is done via command-line scripts (found in `fairseq_cli/`). The training loop and data processing code is unified when possible. As a consequence the code is generally a bit more abstract, you cannot look at just one file if you want to read and understand the training loop.
The codebase can be seen as split into four different parts.
- `fairseq_cli/` contains CLI scripts for data processing, model training and evaluation.
- `fairseq/` (except for `/tasks/`) is like a library of common models, data processing or utility classes and functions
- `fairseq/tasks/` has files which contain code specific to a modeling task
- `examples/` has the documentation and scripts for training and evaluating models related to facebook's papers and corpuses

When training you pass a "task" argument (when calling the CLI scripts) which ensures the appropriate code is run for whatever task you are doing. Many modifications to a task, like using a different model, can done by changing a command-line argument or using a config file to redefine a default (the config is based on hydra+omegaconf). Models defined in fairseq (`fairseq/models/`) will have a decorator enabling you to specify it using one keyword (making it easy to change).

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

One uses `fairseq-hydra-train` to train a model when one has a config and `fairseq-train` if you're willing to specify all the arguments on the commandline. This is an example call to finetune a wav2vec model:
```
fairseq-hydra-train task.data=/work2/rudolf/wav2vec/ch-de/22-08-05/manifest/ model.w2v_path=/work1/rudolf/wav2vec_model.pt checkpoint.save_dir=/work2/rudolf/wav2vec/ch-de/22-08-05/model-a --config-dir $PWD --config-name finetuning.yaml
```
Most of the options are defined inside `finetuning.yaml` , for example the task, this looks like:
```
task:
  _name: audio_finetuning
  data: ???
  normalize: true 
  labels: bpe
```
The name specifies which task defined in  `fairseq/tasks/` will be used, all other options are specific to that. In this case they specify that bpe labels will be used for training and the audio files will be normalized after loading. `???` means the user should specify this, which we did in the above commandline with `task.data=...`.

In `fairseq/examples/` one can always find the commandline necessary to train a recipe.

# speechbrain

[Speechbrain](https://github.com/speechbrain/speechbrain) is for speech processing tasks, they have an impressive array of examples where they get good results (`recipes/`). 

The recipe implementations are less unified than those in fairseq, but there is still a lot of shared code. The core training loop is unified in a `Brain` class (found in `speechbrain/core.py`), the data processing code is specified in each recipe.

Speechbrain uses a powerful yaml config ([HyperPyYaml](https://github.com/speechbrain/HyperPyYAML)) which is used not only to define hyperparameters but also to define data processing related things like the data loading sampler, the loss, the optimizer and the model. A simple one like an 3-layer MLP you could define right in the config like this:
```
enc: !new:speechbrain.lobes.models.VanillaNN.VanillaNN
   input_shape: [null, null, 1024]
   activation: torch.nn.LeakyRelu
   dnn_blocks: 2
   dnn_neurons: 1024
```
A complicated model would be defined by refering to the speechbrain implementation (for example a transformer) for example:
```
encoder: !new:speechbrain.lobes.models.transformer.Transformer.TransformerEncoder
	d_model: 768
	num_layers: 12
	nhead: 8
	d_ffn: 3072
	dropout: 0.1
```
Note that the class will be instantiated when the config is loaded, so you don't have to write any code to do that. To use a different model you can just modify the config.

Sometimes speechbrain's models (e.g. the VanillaNN above) use an `input_shape` to allow them to infer the output shape (by running the model). This is implemented by having those models inherit from `speechbrain.nnet.containers.Sequential` ([code link](https://github.com/speechbrain/speechbrain/blob/424e7921531b0ea6523557ef0fd6ca249936bd26/speechbrain/nnet/containers.py#L18)) which has an `append` method which you use when successively adding layers (this will then pass the input shape and check it works).

The `Brain` class  is initialized with this yaml config, and will hold a reference to the model, the loss, the optimizer, but not the dataset/dataloaders. In contrast to other toolkits speechbrain defines much more in the config.

How the model is used is specified by overriding two methods (of the `Brain` class) `compute_forward` and `compute_objectives`. The former defines how to get from the data to the outputs you need to compute the loss. You then use the latter to define how to compute the loss from those outputs (it's called automatically after the former). Sometimes the implementation will be extremely simple: In `compute_forward` you call the model and in `compute_objectives` you call the loss defined in your config.

The data is typically stored in CSV format (with filepaths pointing to audio data) which you load to a `DynamicItemDataset` like:
```
train_data = sb.dataio.dataset.DynamicItemDataset.from_csv(csv_path=hparams["train_csv"])
```
in your recipe you then specify (via a custom "pipeline" construct) how to go from the data in the CSV to the data you want available in a batch. This can look like this ([code link](https://github.com/speechbrain/speechbrain/blob/develop/recipes/LibriSpeech/ASR/CTC/train_with_wav2vec.py#L234)):
```
@sb.utils.data_pipeline.takes("wav")
@sb.utils.data_pipeline.provides("sig")
def audio_pipeline(wav):
	sig = sb.dataio.dataio.read_audio(wav)
	return sig
```
or this:
```
# 3. Define text pipeline:
@sb.utils.data_pipeline.takes("wrd")
@sb.utils.data_pipeline.provides("wrd", "tokens_list", "tokens_bos")
def text_pipeline(wrd):
	yield wrd
	tokens_list = tokenizer.encode_as_ids(wrd)
	yield tokens_list
	tokens_bos = torch.LongTensor([hparams["bos_index"]] + (tokens_list))
	yield tokens_bos
```

You attach it to the `DynamicItemDataset`:
```
sb.dataio.dataset.add_dynamic_item(train_data, text_pipeline)
sb.dataio.dataset.add_dynamic_item(train_data, audio_pipeline)
```
The "takes" are columns available in the CSV file. The "provides" you can access in `compute_forward`  like this ([code link](https://github.com/speechbrain/speechbrain/blob/develop/recipes/LibriSpeech/ASR/CTC/train_with_wav2vec.py#L35)):

```
def compute_forward(self, batch, stage):
	batch = batch.to(self.device)
	wavs, wav_lens = batch.sig
	tokens_bos, _ = batch.tokens_bos
```
Some magic that happens is when the batch is created padding is automatically done and therefore `batch.sig` also returns a second tensor which specifies the relative length of each sample (relative to the longest sample, which will have the relative length of 1.0).

I like the config and how powerful it is.  I think they've made some good decisions about how to allow people to make the changes they need to write code specific to their problem. I have though sometimes been bitten by behaviour that I did not expect. It definitely is a framework, something that expects you to do things in a certain way. I have also come across a number of bugs and the code is very object oriented. I think it's kind of similar to pytorch lightning (the `Brain` class is similar to the `LightningModule`).

Everything under `speechbrain/` is like a library of everything you could need, recipes will always use models, dataloaders and utility functions from there.

To train a model you call `Brain.fit`, which looks like this ([code link](https://github.com/speechbrain/speechbrain/blob/b1934fa38d9a073eb105e6ec9ffa2119cd4142bf/speechbrain/core.py#L1075)) (all code snippets are shortened):
```
if not (
	isinstance(train_set, DataLoader)
	or isinstance(train_set, LoopedLoader)
):
	train_set = self.make_dataloader(train_set, stage=sb.Stage.TRAIN, **train_loader_kwargs)
[..]
self.on_fit_start()
[..]
for epoch in epoch_counter:
	self._fit_train(train_set=train_set, epoch=epoch, enable=enable)
	self._fit_valid(valid_set=valid_set, epoch=epoch, enable=enable)
[..]
```
There is some logic for creating the dataloader if necessary and then the training loop is started. 
`_fit_train` ([code link](https://github.com/speechbrain/speechbrain/blob/b1934fa38d9a073eb105e6ec9ffa2119cd4142bf/speechbrain/core.py#L982)) encapsulates training for one epoch:
```
[..]
self.on_stage_start(Stage.TRAIN, epoch)
[..]
with tqdm(train_set, initial=self.step, dynamic_ncols=True, disable=not enable) as t:
	for batch in t:
		[..]
		loss = self.fit_batch(batch)
		self.avg_train_loss = self.update_average(loss, self.avg_train_loss)
		[..]
self.on_stage_end(Stage.TRAIN, self.avg_train_loss, epoch)
```
`fit_batch` ([code link](https://github.com/speechbrain/speechbrain/blob/develop/speechbrain/core.py#L842)), it's here that `compute_forward` and `compute_objectives` are used and the optimization step is taken:
```
should_step = self.step % self.grad_accumulation_factor == 0
# Managing automatic mixed precision
if self.auto_mix_prec:
	[..]
else:
	outputs = self.compute_forward(batch, Stage.TRAIN)
	loss = self.compute_objectives(outputs, batch, Stage.TRAIN)
	with self.no_sync(not should_step):
		(loss / self.grad_accumulation_factor).backward()

	if should_step:
		if self.check_gradients(loss):
			self.optimizer.step()
		self.optimizer.zero_grad()
		self.optimizer_step += 1
self.on_fit_batch_end(batch, outputs, loss, should_step)
return loss.detach().cpu()
```

Each recipe in `speechbrain/recipes/` contains a `train*.py` that you use for calling training (there is no single binary like with fairseq). A training command will look like (from the recipe dir):
```
python train_with_wav2vec.py hparams/train_with_wav2vec.yaml
```
or
```
python train_speaker_embeddings.py hparams/train_x_vectors.yaml
```

# k2/icefall/lhotse

This is by far the newest of the three tools I'm comparing here. I wrote k2/icefall/lhotse because there are three different repositories you would make use of: [k2](https://github.com/k2-fsa/k2) for ragged tensors, lattice related operations and loss functions; [lhotse](https://github.com/lhotse-speech/lhotse) for everything related to data such as downloading a corpus, feature extraction and dynamic batching; [icefall](https://github.com/k2-fsa/icefall) for the recipes (for example for transducer models trained on LibriSpeech) which use lhotse and k2.

There is no unified training loop, each recipe will have its own straight-forward implementation making use of lhotse and k2 to do training and inference.

lhotse has a commandline interface which is quite powerful. It lets you extract features, get duration statistics and convert kaldi style data dirs to lhotse's format among other things. See [lhotse documentation](https://lhotse.readthedocs.io/en/latest/cuts.html) for an explanation on lhotse's data constructs like Manifests, Cuts (basically an utterance) and CutSets. An example for getting duration statistics of a CutSet manifest: 
```
$ lhotse cut describe cuts_example.jsonl.gz
Cuts count: 16028
Total duration (hh:mm:ss): 17:53:29
Speech duration (hh:mm:ss): 17:53:29 (100.0%)
Duration statistics (seconds):
mean    4.0
std     4.1
min     0.2
25%     1.4
50%     2.6
75%     5.1
99%     20.5
99.5%   23.3
99.9%   27.6
max     30.5
Recordings available: 16028
Features available: 16028
Supervisions available: 16028
```

You can also do all this via code of course. Here an example, this converts from a kaldi style datadir to a recording and supervision set (lhotse constructs), creates a CutSet (equivalent to a set of utterances), extracts features and outputs a manifest (lhotse's data format):
```
recording_set, supervision_set, _ = load_kaldi_data_dir(kdata, 16000, num_jobs=4)
cuts = CutSet.from_manifests(recordings=recording_set, supervisions=supervision_set)

cuts = cuts.compute_and_store_features(Fbank(), storage_path=outf + '-feats', num_jobs=6, storage_type=LilcomChunkyWriter)

cuts.to_file(outf + '.jsonl.gz')
```


I think lhotse has a very nice interface and definitely plan on basing all my speech recipes around it. I will say though the lhotse codebase has a lot of indirection going on (decorators everywhere) making it hard to reason about the code and debugging sometimes a pain.

k2 is more low-level. It's for example used for [RaggedTensors](https://k2-fsa.github.io/k2/core_concepts/index.html#ragged-arrays) like this where y is a list of lists:
```
y = k2.RaggedTensor(y)
```
And for FSTs. k2 saves FSTs in pytorch's (actually pickling) format so you load them from disk like this:
```
dct = torch.load(args.decode_graph, map_location=device)
decoding_graph = k2.Fsa.from_dict(dct)
```
Otherwise it contains many constructs for loss functions, FST operations and decoders: `k2.ctc_loss`, `k2.rnnt_loss`, `k2.rnnt_loss_pruned`, `k2.intersect`, `k2.top_sort`, `k2.RnntDecodingStream`  are some examples. You will only use it yourself if you want to do something with the loss function, the decoder or an FST. It's not necessary to understand if you just want to run recipes and change the data and/or model.

Now about icefall. Regarding the repo layout, `egs/` is for the recipes (don't understand why they didn't move away from the opaque term kaldi that used) and `icefall/` basically contains utility functions. Note, unlike in speechbrain and fairseq, this dir `icefall/` (the dir where all the python library code goes) contains not much code as the majority is kept recipe specific, for example the training loop. This makes the recipes easy to read and modify as there is no framework involved (it's just normal pytorch) and minimal coupling: You don't have to jump to ten different places to follow the logic.

So for example [egs/librispeech/ASR/pruned_transducer_stateless2](https://github.com/k2-fsa/icefall/tree/master/egs/librispeech/ASR/pruned_transducer_stateless2) contains the files `beam_search.py`, `conformer.py`, `decode.py`, `decoder.py`, `joiner.py`, `model.py`, `optim.py`, `scaling.py` and `train.py`. Basically everything is in there, and only the "boring" stuff (like word symbol tables, checkpointing, logging) is relegated to the `icefall/` library.

Sidenote, there's some interesting new ideas regarding the basics of DNNs in icefall. The main one is that instead of using LayerNorm/BatchNorm it makes sense to have a differentiable scale parameter for the weight matrices. The `scaling.py` file contains new implementations of all basic modules that follow this new approach and `optim.py` has a custom optimizer for these new modules.

Back to the main thread: Everything important is inside the recipe folder. The training loop could not look more normal (that's good!). After the usual setup ([code link](https://github.com/k2-fsa/icefall/blob/master/egs/librispeech/ASR/pruned_transducer_stateless2/train.py#L987)):
```
[..]
for epoch in range(params.start_epoch, params.num_epochs):
	[..]
	train_one_epoch(params=params, model=model, optimizer=optimizer, scheduler=scheduler, sp=sp, train_dl=train_dl, valid_dl=valid_dl, scaler=scaler,
tb_writer=tb_writer, world_size=world_size, rank=rank)
	[..]
	save_checkpoint(params=params, model=model, optimizer=optimizer, scheduler=scheduler, sampler=train_dl.sampler, scaler=scaler, rank=rank)
```
`train_one_epoch` looks like ([code link](https://github.com/k2-fsa/icefall/blob/master/egs/librispeech/ASR/pruned_transducer_stateless2/train.py#L760)):
```
for batch_idx, batch in enumerate(train_dl):
	[..]
	loss, loss_info = compute_loss(params=params, model=model, sp=sp, batch=batch,
is_training=True, warmup=(params.batch_idx_train / params.model_warm_step))
	[..]
	if batch_idx > 0 and batch_idx % params.valid_interval == 0:
		valid_loss = compute_validation_loss(params=params, model=model, sp=sp,
			valid_dl=valid_dl, world_size=world_size)
	[..]
```

One neat feature is that if a training crashes or you ctrl+c it, the current batch will automatically be dumped to disk for you to inspect it.

I like the overall structure, and appreciate the focus on delivering something first and thinking about unifying it later. The results are very good.
However, because this is all very new it's in a state of flux. Things change and sometimes there are bugs. On the bright side everyone involved is very responsive and if something doesn't work one can get help very quickly.

Training is similar to speechbrain with the recipe specific `train.py` files. Unfortunately there are not many READMEs, but the top of a file usually has documentation. For "egs/librispeech/ASR/pruned_transducer_stateless2" the training command looks like:
```
PYTHONPATH=/path/to/icefall ./pruned_transducer_stateless2/train.py \
--world-size 4 \
--num-epochs 30 \
--start-epoch 0 \
--use-fp16 1 \
--exp-dir pruned_transducer_stateless2/exp \
--full-libri 1 \
--max-duration 550
```

Note setting the `PYTHONPATH` is necessary as icefall is not pip-installable, and instead one uses the `PYTHONPATH` so that the imports (`from icefall.utils import bla`) in `train.py` work.
