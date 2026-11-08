# Training guide

How to build a Brieno model from raw nanopore reads.

If you only want to *run* a trained model — basecalling, demultiplexing, species
identification, modification calling — you do not need any of this. See
[Downstream tasks](../getting-started/tasks.md).

Every command below has its own `--help` with the full list of flags and their
defaults; this guide covers the workflow and the ideas behind it, not the flag table.
For the flags, see the [CLI reference](../reference/cli.md).

## The pipeline

```
raw FAST5/POD5
      │
      │  brieno train prep              ✔ implemented
      ▼
  signal chunks (.npy shards)
      │
      │  brieno train merge             ✔ implemented
      ▼
  one .npy per dataset
      │
      │  brieno train tokenizer         ✔ implemented
      ▼
  signal tokenizer
      │
      │  brieno train encode            ✔ implemented
      ▼
  tokenized corpus
      │
      │  brieno train transformer       ✔ implemented
      ▼
  token transformer
      │
      │  task fine-tuning               ○ not yet implemented
      ▼
  basecalling / demux / species / modification heads
```

Everything up to a pretrained transformer is here. Fine-tuning a task head is being
prepared for a later release; until then the task models are published as weights.

| Step | Command | Page |
|------|---------|------|
| 1. Chunk the raw signal | `brieno train prep` | [Data preparation](data-preparation.md#1-chunk-the-raw-signal) |
| 2. Merge shards per dataset | `brieno train merge` | [Data preparation](data-preparation.md#2-merge-the-shards-into-one-file-per-dataset) |
| 3. Pretrain the tokenizer | `brieno train tokenizer` | [Tokenizer](tokenizer.md) |
| 4. Encode the corpus | `brieno train encode` | [Tokenizer](tokenizer.md#4-encode-the-corpus-into-tokens) |
| 5. Pretrain the transformer | `brieno train transformer` | [Transformer](transformer.md) |
| 6. Fine-tune a task head | — | [Fine-tuning](fine-tuning.md) |

## Requirements

Install Brieno as described in [Installation](../getting-started/installation.md).
Training needs a CUDA GPU. The CRF parts of the downstream tasks additionally need
`ont-koi` (x86_64) or `cupy` (any CUDA platform) — not required for the tokenizer.
