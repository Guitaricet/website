+++
title="A Small TPU Guide"
date=2023-07-17
slug="tpus"
description="A guide to using TPUs with Levanter with a few general tips and tricks"
+++

[TPU research cloud](https://sites.research.google/trc/about/) is an amazing resource to get a scientific grant for a lof of compute.
However, working with TPUs could be complicated and the official PyTorch support for TPUs is pretty terrible.
In this guide, we'll share some of the practices of using TPUs.

<!-- more -->

When you join the program, typically you get something like this:

> Thanks again for your interest in using Cloud TPUs to accelerate your machine learning research.
  Your Google Cloud project tokyo-carving-231908 now has access to the following quota free of charge for 60 days:
>
> **5 on-demand Cloud TPU v3-8 device(s) in zone europe-west4-a**
> 
> **100 preemptible Cloud TPU v2-8 device(s) in zone us-central1-f**
> 
> **5 on-demand Cloud TPU v2-8 device(s) in zone us-central1-f**

Note that typically, you only get the devices with eight TPU chips.
These are the smallest TPU devices, so you might not be able to scale beyond 1B params or so,
because there is no straightforward way to use multiple separte TPU devices in a single run.

## Levanter

[Levanter](https://github.com/stanford-crfm/levanter) is an LLM training framework by Stanford Center for Research on Foundation Models.
Please refer to their documentation on GitHub.

Here's a rough pipeline of running TPU jobs using Levanter.
Note that these commands are executed on your machine, not on the TPU instance.
They will create TPUs, copy the script to all of the TPU workers, and run it.
Commands assume you have [Google Cloud CLI](https://cloud.google.com/sdk/docs/install) (`gcloud`) installed and you are authenticated.

> Even when not using Levanter, we recommend to use `gcloud` instead of the web interface for TPUs.
> We found that the web interface a bit confusing, plus as of July 2023 it gives you `UnknownError` frequently instead of providing the exact error as `gcloud` does.

```bash
git pull https://github.com/stanford-crfm/levanter
cd levanter

# Define your instance name, GCP zone, and TPU type
export NAME=levanter1
export ZONE=us-central1-f
export TPU_TYPE=v2-8

# Create a TPU VM instance (this might take a minute)
bash infra/spin-up-vm.sh $NAME -z $ZONE -t $TPU_TYPE

# Run training
# You need to provide the correct --config_path of your run
# and to provide a GCP Storage bucket path for your checkpoints
gcloud compute tpus tpu-vm ssh \
    $NAME --zone $ZONE --worker=all \
    --command 'WANDB_API_KEY=<YOUR KEY> levanter/infra/run.sh python levanter/src/levanter/main/train_lm.py --config_path levanter/config/gpt2_small.yaml --trainer.checkpointer.base_path gs://your-levanter-gcp-bucket'
```

This will start your training.
In case of preemptible TPUs, Levanter has [additional support for automatic restarts](https://github.com/stanford-crfm/levanter/blob/main/docs/Getting-Started-TPU-VM.md#using-the-babysitting-script-with-a-preemptible-or-trc-tpu-vm).

**From Levanter documentation:**
You can run it like this:

```
infra/babysit-tpu-vm <name> -z <zone> -t <type> [--preemptible] -s infra/setup-tpu-vm-nfs.sh -- \
    WANDB_API_KEY=... levanter/infra/run.sh python levanter/src/levanter/main/train_lm.py --config_path levanter/config/gpt2_small.yaml
```

That `--` is important! It separates the spin up args from the running args.
Also, you should never use `launch.sh` with babysit, because nohup exits immediately with exit code 0.

**Updating code on the TPUs**

If you want to make a modification to a config or the soruce code, you can synchronize your local Levanter with the tpus using `scp`.

```bash
cd ..
gcloud compute tpus tpu-vm scp --recurse levanter/ $NAME:/home/vlialin/ --zone $ZONE --worker=all
```

It might take some time and it's best to just provide it just a config directory or a specific file you modified:

```bash
gcloud compute tpus tpu-vm scp --recurse configs levanter1:/home/vlialin/levanter/ --zone $ZONE --worker=all
```

## Levanter Configs

Levanter suppports using HuggingFace datasets for training. Dataset config looks like this:
```
id: text-machine-lab/constrained_language
cache_dir: "gs://vlialin-levanter/dataset_cache/minibert"
text_key: TEXT
```

Where `id` is the Huggingface Dataset name, `cache_dir` is a path to your GCP bucket to keep the cache of the pre-processed dataset,
and `text_key` is the name of the dataset key of the text that you want to train on.

If you need to do more complex preprocessing, I recommend to either do it offline and upload to Huggingface or to modity the Levanter source code.

## TPU Performance Guide

Unfortunately, there is no such thing as `nvidia-smi` or `nvitop` for TPUs.
This is largely because TPUs are designed differently than GPU: they work on a model where the entire model must fit in the TPU's memory,
and computations are heavily optimized around this.
This can make tracking memory utilization a bit less straightforward.

The closest you can get to `nvidia-smi` is [jax-smi](https://github.com/ayaka14732/jax-smi).
Please follow their instructions, they require some minimal code modifications.

As far as I know, you should think that any kind of profiling **adds overhead** and should be performed separate to the main trainig run.

If you want to dig deeper into TPU profiling, here's a few guides:
1. [Jax profiler](https://jax.readthedocs.io/en/latest/profiling.html)
2. [Cloud TPU performance guide](https://cloud.google.com/tpu/docs/performance-guide)

---
Work in progress. To be updated
