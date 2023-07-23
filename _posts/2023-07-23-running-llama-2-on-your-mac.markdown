---
layout: post
title: >
  Running LLaMA 2 models on your Mac
date: 2023-07-23 12:00:00 +0000
---

![Running LLaMA 2 model in llama.cpp chat](/content/images/llama-chat.png)

LLMs are the rage nowadays. Meta recently made [LLaMA 2 model weights](https://about.fb.com/news/2023/07/llama-2/) available for commercial use ([under some conditions](https://daringfireball.net/linked/2023/07/20/facebook-open-llama-2)), which means that mere mortals (like me) got access to building cool "AI"-y stuff without owning a personal data center.

To get a sense of how to interact with the model, I wanted to build and run it on my machine first. I don't have a beefy workstation with chonky GPU-s in it, but thanks to [Georgi Gerganov](https://github.com/ggerganov), I don't need to: [llama.cpp](https://github.com/ggerganov/llama.cpp).

There is a good [tutorial](https://gist.github.com/cedrickchee/e8d4cb0c4b1df6cc47ce8b18457ebde0) by [Cedric Chee](https://gist.github.com/cedrickchee) on converting and running LLaMA (1) models, but some steps are missing for LLaMA 2 or are out of date. Hopefully, this note will save you (and future me) a little bit of time.

## Building llama.cpp

For compiling C++ code, you will need to install command line tools:

```sh
xcode-select --install
```

Clone the repo:

```sh
git clone https://github.com/ggerganov/llama.cpp.git
```

Next, build llama.cpp

```sh
# jump into llama.cpp
cd llama.cpp

# using Metal allows the computation to be executed on the GPU for Mac
LLAMA_METAL=1 make
```

## Downloading LLaMA 2 models and converting them

You will need to fill out a [form](https://ai.meta.com/resources/models-and-libraries/llama-downloads/) to get download links to models. Next, we will optimize them using llama.cpp. Copy downloaded files under `llama.cpp/models`. I have downloaded `llama-2-7b` (the smallest model) for starters.

```sh
# copy models from llama to llama.cpp
cp tokenizer.model tokenizer_checklist.chk $PATH_TO_LLAMA_CPP_REPO/models/

# copying the entire directory to avoid mess under llama.cpp/models
cp -r llama-2-7b/ $PATH_TO_LLAMA_CPP_REPO/models/llama-2-7b/
```

The conversion script is written in Python and requires extra dependencies. To avoid polluting the global environment, let's create a virtual one:

```sh
# create a virtual env under llama.cpp
python3 -m venv llamaenv

# activate the environment
source llamaenv/bin/activate

# install python deps
python3 -m pip install -r requirements.txt

# convert models
python3 convert.py models/llama-2-7b/
```

Once done, you can deactivate the virtual environment by typing `deactivate`. Now we need to quantize the model: convert the model weights from 16-bit floating-point numbers to 4-bit integers.


```sh
# quantize the model to 4-bits (using q4_0 method)
./quantize ./models/llama-2-7b/ggml-model-f32.bin models/llama-2-7b/ggml-model-q4_0.bin q4_0
```

## Running queries against the model

A note on memory/disk requirements from llama.cpp [documentation](https://github.com/ggerganov/llama.cpp#memorydisk-requirements):

> "As the models are currently fully loaded into memory, you will need adequate disk space to save them and sufficient RAM to load them. At the moment, memory and disk requirements are the same".

I was able to run a 7B model on my MacBook Air with 16GB of RAM without a problem.

Running the model is as simple as:

```sh
# -t 8: number of threads to use during computation.
# -n 128: the number of tokens to predict.
# -p 'Is avocado a vegetable?': the prompt.

./main -m ./models/llama-2-7b/ggml-model-q4_0.bin -t 8 -n 128 -p 'Is avocado a vegetable?'
```

**Bonus**: `llama.cpp` comes with a simple web chat application similar to ChatGPT. You can run it by pointing at the model:

```sh
# -c 2048: size of the prompt context

./server -m models/llama-2-7b/ggml-model-q4_0.bin -c 2048
```
