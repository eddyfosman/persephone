# Persephone v0.1.3

Persephone (/pərˈsɛfəni/) is an automatic phoneme transcription tool. Traditional speech recognition tools require a large pronunciation lexicon (describing how words are pronounced) and much training data so that the system can learn to output orthographic transcriptions. In contrast, Persephone is designed for situations where training data is limited, perhaps as little as 30 minutes of transcribed speech. Such limitations on data are common in the documentation of low-resource languages. It is possible to use such small amounts of data to train a transcription model that can help aid transcription, yet such technology has not been widely adopted.

The goal of Persephone is to make state-of-the-art phonemic transcription accessible to people involved in language documentation. Creating an easy-to-use user interface is central to this. The user interface and APIs are currently a work in progress.

The tool is implemented in Python/Tensorflow with extensibility in mind. Currently just one model is implemented, which uses bidirectional LSTMs and the connectionist temporal classification (CTC) loss function.

We are happy to offer direct help to anyone who wants to use it. If you're having trouble, contact Oliver Adams at oliver.adams@gmail.com. We are also very welcome to thoughts, constructive criticism, help with design and development and any pull requests you may have.

## Quickstart

This guide is written to help you get the tool working on your machine. We will
use a example setup that involves training a phoneme transcription tool
for [Yongning Na](http://lacito.vjf.cnrs.fr/pangloss/languages/Na_en.php). For
this we use a small (even by language
documentation standards) sub-sampling of elicited speech of
Yongning Na, a language of Southwestern China.

The example that we will run can be run on most personal computers without a
graphics processing unit (GPU), since I've made the settings less
computationally demanding than it would be for optimal transcription quality.
Ideally you'd have access to a server with more memory and a GPU, but this
isn't necessary.

The code has been tested on Mac and Linux systems. It hasn't yet been tested on Windows.

For now you must open up a terminal to enter commands at the command line. (The
commands below are prefixed with a "$". Don't enter the "$", just whatever
comes afterwards).

### 1. Installation

#### Installation option 1: Using the Docker container

To simplify setup and system dependencies, a Docker container has been created.
This just requires [Docker to be installed](https://docs.docker.com/install/).
Once you have installed docker you can fetch our container with:

```
$ docker pull oadams/persephone
```

Then run it in interactive mode:

```
$ docker run -it oadams/persephone
```

This will place you in an environment where Persephone and its
dependencies have been installed, along with the example Na data.

#### Installation option 2: A "native" install

Ensure Python 3 is installed.

You will also need to install some system dependencies. For your convienence we
have an install script for dependencies for Ubuntu. To install the Ubuntu
binaries, run `./bootstrap_ubuntu.sh` to install ffmpeg packages. On MacOS we
suggest installing via Homebrew with `brew install ffmpeg`.

We now need to set up a virtual environment and install the library.

```
$ python3 -m virtualenv -p python3 persephone-venv
$ source persephone-venv/bin/activate
$ pip install -U pip
$ pip install git+git://github.com/oadams/persephone.git
```

(This library can be installed system-wide but it is recommended to install in a virtualenv.)

I've uploaded an example dataset that includes some Yongning Na data that has already been preprocessed. We'll use this example dataset in this tutorial. Once we confirm that the software itself is working on your computer, we can discuss preprocessing of your own data.

Create a working directory for storage of the data and running experiments:

```
mkdir persephone-tutorial/
cd persephone-tutorial/
mkdir data
```

Get the data [here](https://cloudstor.aarnet.edu.au/sender/?s=download&token=b6789ee3-bbcb-7f92-2f38-18ffc1086817)

Unzip `na_example.zip`. There should now be a directory `na_example/`, with
subdirfectories `feat/` and `label/`. You can put `na_example` anywhere, but
for the rest of this tutorial I assume it is in the working directory: `persephone-tutorial/data/na_example/`.

### 2. Training a toy Na model

One way to conduct experiments is to run the code from the iPython interpreter. Back to the terminal:

```
$ ipython
> from persephone import corpus
> corp = corpus.ReadyCorpus("data/na_example")
> from persephone import run
> run.train_ready(corp)
```

You'll should now see something like:

```
3280
2018-01-18 10:30:22.290964: I tensorflow/core/platform/cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: SSE4.1 SSE4.2 AVX AVX2 FMA
    exp_dir ../exp/1, epoch 0
        Batch 0
        Batch 1
        ...
```
 
The message may vary a bit depending on your CPU, but if it says "Batch 0" at the bottom without an error, then training is very likely working. Contact me if you have any trouble getting to this point, or if you had to deviate from the above instructions to get to this point.

On the current settings it will train through batches 1 to 200 or so for at least 10 "epochs", potentially more. If you don't have a GPU then this will take quite a while, though you should notice it converging in performance within a couple hours on most personal computers.

After a few epochs you can see how its going by going to opening up
`exp/<experiment_number>/train_log.txt`. This will show you
the error rates on the training set and the held-out validation set. In the
`exp/<experiment_number>/decoded` subdirectory, you'll see the validation set reference in `refs` and the model hypotheses for each epoch in `epoch<epoch_num>_hyps`.

Currently the tool assumes each utterance is in its own audio file, and that for each utterance in the training set there is a corresponding transcription file with phonemes (or perhaps characters) delimited by spaces.

### 3. Using your own data.

If you have gotten this far, congratulations! You're now ready to start using
your own data. The example setup we created with the Na data illustrates a
couple key points, including how your data should be formatted, and how you
make the system read that data. In fact, if you format your data in the same
way, you can create your own Persephone `Corpus` object with:

```
corp = corpus.ReadyCorpus("<your-corpus-directory>")
```

We describe the specifics below.

#### Formatting your data

Interfacing with data is a key bottleneck in useability for speech recognition
systems. Providing a simple and flexible interface to your data is currently the
most important priority for Persephone at the moment. This is a work in
progress.

Current data formatting requirements:
	* Audio files are stored in `your-corpus/feat/`. A wide variety of audio
	formats are supported.
	* Transcriptions are stored in text files in `your-corpus/label/`
	* Each audio file is short (ideally no longer than 10 seconds). There is a
	script (`persephone/scripts/split_eafs.py`) added by Ben Foley to split
	audio files into utterance-length units based on ELAN input files.
	* Each audio file in `feat/` has a corresponding transcription file in
	`label/` with the same *prefix* (the bit of the filename before the
	extension). For
	example, if there is `feat/utterance_one.wav` then there should be
	`label/utterance_one.<extension>`. `<extension>` can be whatever you want,
	but it should describe how the labelling is done. For example, if it is
	phonemic then `feat/utterance_one.phonemes` is a meaningful filename.
	* Each transcript file includes a space-delimited list of *labels* to
	the model should learn to transcribe. For example, in
	`data/na_example/label/crdo-NRU_F4_ACCOMP_PFV.0.phonemes` contains
		* l e dz ɯ z e l e dz ɯ z e
	while data/na_example/label/crdo-NRU_F4_ACCOMP_PFV.0.**phonemes_and_tones**
	might contain:
		* l e ˧ dz ɯ ˥ z e ˩ | l e ˧ dz ɯ ˥ z e ˩
	Persephone is agnostic to what your chosen labels are. It simply tries to
	figure out how to map speech to that labelling. These labels can be
	multiple characters long: the spaces demarcate labels. They can be any
	unicode characters.

#### On choosing an appropriate label granularity.

#### On creating validation and test sets.
