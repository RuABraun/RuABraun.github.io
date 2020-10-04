---
layout: post
title:  "First post: Ark and scp files in kaldi"
date:   2020-10-04 14:40:02 +0200
categories: jekyll update
---

Both file types are organized by having keys and values for each key.

# Scp files

Scp files, usually ending with the ".scp" suffix, always reserve the first field in a line (a field is a word separated by spaces, you could also call it column) for a key (usually an utterance ID). The rest of the line is treated as a pointer to data (is the value). 
What this means is that the scp file never contains any data, it just contains something which points to it (the data). See for example the wav.scp file which will look something like this usually:

```
wav-id-1 /path/to/file-1.wav
wav-id-2 /path/to/file-2.wav
wav-id-3 /path/to/file-3.wav
```

It can be a file path, but can also be more complex like:
```
wav-id-1 sox /path/to/file-1.wav -r 8k - |
```
Kaldi's code is written so that it will recognize and execute the above command (starting from after the key), which results in the wav file being read with a sampling rate of 8kHz. This is a convenient way of doing the resampling on the fly (instead of resampling and having to waste space storing it somewhere).

Another file one will see a lot when using kaldi is the `feats.scp` file which looks like:
```
utt-id-1 /path/to/file-1.ark:44
utt-id-2 /path/to/file-1.ark:760
utt-id-3 /path/to/file-1.ark:1520
```
The number after the `:` is a byte offset.

# Ark files 

Instead of the value being a pointer to the data, in ark files the value **is** the data. Usually this means it is in binary format, so you can't visually inspect it, however kaldi has binaries for converting from binary to text format. So for example take the ark file from the feats.scp example above and call:

```
copy-feats ark:/path/to/file-1.ark ark,t:/path/to/file-1.ark.txt
```

Note the `ark,t` means that the values in the output will be written in textual format, so you can open the file and see what the values for the (example) MFCC coefficients are for each frame.

Another example of an ark style file, although it doesn't have the ".ark" suffix, is the `text` file you will find in kaldi data folders. The first field/column is the utterance ID, the rest of the line is the words belonging to that utterance. The words are the data.

---

Kaldi has a script for subsetting `scp` files you may find useful called `utils/subset_scp.pl`.

When you call kaldi binaries you often have to prefix with `ark:` or `scp:` to tell kaldi what type of file to expect. You can then mix types, for example I could use `utils/subset_scp.pl` to subset `feats.scp` and create a file called `feats_subset.scp`, and then copy a subset of the feature data by calling
```
copy-feats scp:feats_subset.scp ark:copied_feats_subset.ark
```
Notice how the input argument is `scp`, so kaldi knows the input file is pointing to the data I want to use, and the output is `ark`, so kaldi knows to actually write the data pointed to to a new file.

It's not possible to create just a `scp` file from an `ark` file, you have to create both the `ark` and `scp` files. So you have to do something like this:
```
copy-feats scp:/path/to/feats.ark ark,scp:copied_feats.ark,copied_feats.scp 
```

