# CLI reference

```
brieno [-h] [-v] <command> ...
```

| Option | Description |
|--------|-------------|
| `-h, --help` | Show help and exit. |
| `-v, --version` | Show the version and exit. |

Run `brieno <command> --help` for the options of one command.

## Commands

| Command | Purpose | Status |
|---------|---------|--------|
| `brieno train` | Developer tools: data preparation and model training. | Available |
| `brieno basecaller` | Raw signal → basecalls. | Coming soon |
| `brieno demux` | Raw signal → one barcode per read. | Coming soon |
| `brieno classifier` | Raw signal → one species per read. | Coming soon |
| `brieno modcaller` | Raw signal → modified bases. | Coming soon |
| `brieno download` | Fetch model weights. | Coming soon |

## brieno train

Building a Brieno model: data preparation and training. See the
[training guide](../training/index.md) for the workflow.

```
brieno train <subcommand> [options]
```

| Subcommand | Description |
|------------|-------------|
| [`prep`](#brieno-train-prep) | Turn raw FAST5/POD5 into normalised signal chunks. |
| [`merge`](#brieno-train-merge) | Concatenate one dataset's chunk shards into a single `.npy`. |
| [`tokenizer`](#brieno-train-tokenizer) | Pretrain the signal tokenizer on chunked raw signal. |
| [`encode`](#brieno-train-encode) | Turn signal chunks into token ids with a trained tokenizer. |
| [`transformer`](#brieno-train-transformer) | Pretrain the token transformer with masked-token prediction. |

### brieno train prep

Read raw signal, normalise each read by median/MAD, trim the open-pore and stall region
at the read start, and cut the rest into overlapping fixed-length chunks, written as
`float16` `.npy` shards of shape `(N, 1, chunk-size)`.

| Option | Default | Description |
|--------|---------|-------------|
| `-i, --input DIR` | *required* | Directory of `.fast5` / `.pod5` files. |
| `-o, --output DIR` | `pretrain_chunk` next to `--input` | Output directory. |
| `-m, --modality` | `dna_r9` | Sets the trim threshold: `dna_r9`, `dna_r10`, `rna_002`, `rna_004`. |
| `-c, --chunk-size` | `16384` | Samples per chunk. |
| `--overlap` | `2048` | Overlap between chunks, in samples. |
| `-l, --min-length` | `16384` | Skip reads shorter than this many samples. |
| `--max-chunks-per-file` | `100000` | Chunks per output shard. |
| `-r, --recursive` | off | Also look in subdirectories of `--input`. |
| `--resume` | off | Skip input files that already have output shards. |
| `-j, --workers N` | `1` | Process N files in parallel (turns off per-file progress bars). |
| `--no-progress` | off | No per-file progress bar. |

### brieno train merge

Merge the shards `brieno train prep` wrote for one dataset into the single array the
tokenizer trainer reads. Name the output after the dataset: that name is the key used in
the training config.

| Option | Default | Description |
|--------|---------|-------------|
| `-i, --input DIR` | *required* | Directory of `.npy` shards for **one** dataset. |
| `-o, --output FILE` | *required* | Output `.npy` file. |
| `-r, --recursive` | off | Also look in subdirectories of `--input`. |
| `-f, --force` | off | Overwrite an existing output. |
| `--no-progress` | off | No progress bar. |

### brieno train tokenizer

Self-supervised pretraining of the residual-vector-quantization tokenizer. Run under
`torchrun` for multi-GPU training.

| Option | Default | Description |
|--------|---------|-------------|
| `-c, --config JSON` | *required* | Training config (see `configs/` for an example). |
| `-s, --seed` | `42` | Base seed; each rank adds its index. |
| `-n, --run-name NAME` | a timestamp | Output subdirectory under `training.save_dir`. |
| `--resume CKPT` | — | Continue from a checkpoint written by an earlier run. |

Config overrides (leave unset to use the config file):

| Option | Overrides |
|--------|-----------|
| `--lr` | `training.lr` |
| `--batch-size` | `training.batch_size` |
| `--epochs` | `training.epochs` |
| `--commitment-cost` | `model.commitment_cost` |
| `--ema-decay` | `model.ema_decay` |
| `--dropout` | `model.encoder_dropout` and `model.decoder_dropout` together |

### brieno train encode

Run a trained tokenizer over a merged signal file and write the discrete token ids the
transformer pretrains on: `(N, num_quantizers, seq_len)` `uint16`. The tokenizer
checkpoint carries its own architecture, so no config is needed.

| Option | Default | Description |
|--------|---------|-------------|
| `-m, --model CKPT` | *required* | Tokenizer checkpoint from `brieno train tokenizer`. |
| `-i, --input FILE` | *required* | Merged signal `.npy` for one dataset. |
| `-o, --output FILE` | *required* | Output token `.npy`. |
| `-b, --batch-size` | `512` | Batch size. |
| `-j, --num-workers` | `4` | Data-loader workers. |
| `--device` | `cuda` | `cuda`, `cuda:N` or `cpu`. |
| `-f, --force` | off | Overwrite an existing output. |
| `--no-progress` | off | No progress bar. |

### brieno train transformer

Self-supervised pretraining of the transformer on tokenized signal: spans of token
positions are hidden and predicted from their context, in every quantizer layer at once.
Run under `torchrun` for multi-GPU training.

| Option | Default | Description |
|--------|---------|-------------|
| `-c, --config JSON` | *required* | Training config (see `configs/` for an example). |
| `-s, --seed` | `training.seed` | Base seed; each rank adds its index. |
| `-n, --run-name NAME` | a timestamp | Output subdirectory under `training.save_dir`. |
| `--resume CKPT` | — | Continue from a checkpoint written by an earlier run. |

Config overrides (leave unset to use the config file):

| Option | Overrides |
|--------|-----------|
| `--lr` | `training.lr` |
| `--batch-size` | `training.batch_size` |
| `--epochs` | `training.epochs` |
| `--grad-accum-steps` | `training.grad_accum_steps` |
