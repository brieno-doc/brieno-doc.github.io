---
icon: lucide/house
---

# Brieno

**A foundation model for raw nanopore signal through discrete tokenization.**

[![bioRxiv](https://img.shields.io/badge/bioRxiv-2026.11.05.764536-green)](https://www.biorxiv.org/content/10.64898/2026.11.05.764536v1)
[![Python](https://img.shields.io/badge/python-3.10%2B-blue)](https://www.python.org/)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://github.com/guangzhaocs/Brieno/blob/main/LICENSE)
[![PyPI](https://img.shields.io/pypi/v/brieno)](https://pypi.org/project/brieno/)
[![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-Brieno-yellow)](https://huggingface.co/guangzhaocs/Brieno)

## Overview

**Brieno** is a self-supervised model that learns reusable representations of raw
nanopore signals. It first converts raw signals into discrete tokens through residual
vector quantization and then pretrains a Transformer to predict masked tokens from their
context. The pretrained Transformer can be fine-tuned with task-specific heads for
multiple downstream tasks.

![Brieno pipeline: tokenizer pretraining with residual vector quantization, masked-token pretraining of the transformer, and fine-tuning on downstream tasks](assets/model_overview.png)

With frozen or fine-tuned features, the model:

- identifies **species** without requiring reference genomes at inference time;
- achieves higher overall accuracy than Dorado in **native-barcode demultiplexing**;
- approaches Dorado’s **basecalling** performance across six genomes;
- achieves a higher read-level AUROC than Dorado’s 5mC model for **CpG methylation detection**.

## Where to go next

<div class="grid cards" markdown>

-   :lucide-download: **[Installation](getting-started/installation.md)**

    Install Brieno and the extras for your raw-signal format.

-   :lucide-layers: **[Downstream tasks](getting-started/tasks.md)**

    Basecalling, demultiplexing, species identification and modification calling.

-   :lucide-graduation-cap: **[Training guide](training/index.md)**

    Build your own model, from raw FAST5/POD5 to a pretrained transformer.

-   :lucide-terminal: **[CLI reference](reference/cli.md)**

    Every `brieno` command and its options.

</div>

## Name origin

*Brieno* is a shortened form of [*Brienomyrus*](https://en.wikipedia.org/wiki/Brienomyrus), a genus of weakly
electric fish that communicate through patterns of electric pulses. The name echoes Oxford
Nanopore's fish-inspired tool names, such as Bonito and Dorado, and also reflects the
model's aim to learn informative patterns in the ionic current signals recorded during
nanopore sequencing.
