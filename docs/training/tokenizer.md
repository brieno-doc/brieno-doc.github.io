# Tokenizer

Pretrain the signal tokenizer, then use it to turn the corpus into tokens.

## 3. Pretrain the tokenizer

The tokenizer learns to reconstruct raw signal through a residual vector quantizer.
The loss is `L1(reconstruction, signal) + vq_loss` — no labels of any kind, so any
corpus of raw reads will do.

=== "One GPU"

    ```bash
    brieno train tokenizer -c configs/tokenizer_dna_r9.json
    ```

=== "Four GPUs on one node"

    ```bash
    torchrun --nproc_per_node=4 -m brieno.cli train tokenizer -c configs/tokenizer_dna_r9.json
    ```

### The config

Start from
[`configs/tokenizer_dna_r9.json`](https://github.com/guangzhaocs/Brieno/blob/main/configs/tokenizer_dna_r9.json),
which is the recipe of the released R9.4.1 tokenizer. Three sections:

**`datasets`** — where the corpus is and how to mix it.

`pretrain_weights` is a list of `{name, weight}`. Weights are normalised, so they are
proportions of the training stream, *not* of the data on disk: a dataset with a
weight of 0.12 contributes 12 % of the chunks the model sees however large its file
is. This is what keeps a corpus balanced when one organism happens to have ten times
more reads than another.

`validation` is a separate list of dataset names. Each file is split once by a fixed
seed into a training part and a validation part (at most 10 % of the file, capped by
`validation_size_per_dataset`), so the held-out chunks are the same on every run and
on every machine. Listing a dataset in `validation` but not in `pretrain_weights` is
deliberate and useful: it measures generalisation to an organism the model never
trained on.

**`training`** — `epoch_length` is the number of *draws* per epoch, not a pass over
the data: each draw picks a dataset by weight and then a random chunk inside it. This
makes epochs comparable between runs with different corpus sizes. `save_dir` is the
parent; each run gets a subdirectory (`--run-name`, or a timestamp).

**`model`** — `downsample` sets the token rate (8 means 16,384 samples become 2,048
frames), `num_quantizers` × `num_tokens` is the codebook stack, and `quant_dim`
quantises in a projected low-dimensional space rather than the full hidden width.
`commitment_cost` and `ema_decay` control the quantizer; the codebooks are EMA-tracked,
so the optimizer only ever sees the encoder and decoder.

Anything in the config can be overridden on the command line (`--lr`, `--epochs`,
`--commitment-cost`, …), which is convenient for sweeps without writing a config per
run.

### Output

```
<save_dir>/<run name>/
    config.json          the config this run actually used, overrides applied
    history.jsonl        one JSON object per logged step and per epoch
    tokenizer_ep<N>.pt   checkpoint after each epoch
    tokenizer_best.pt    lowest validation loss so far
```

Checkpoints hold a plain tokenizer `state_dict` plus the optimizer, scheduler and
scaler state, so `--resume <checkpoint>` continues a run that hit a wall-clock limit.

!!! tip "On a cluster"

    Write `save_dir` to shared storage rather than node-local disk, or a timed-out job
    leaves you with nothing to resume from.

### Monitoring

Metrics are written to `history.jsonl` rather than an experiment-tracking service, so
a run's history lives next to its weights and can be read without an account:

```bash
python -c "
import json
for line in open('history.jsonl'):
    r = json.loads(line)
    if r['split'] == 'val':
        print(r['epoch'], round(r['recon'], 4), round(r['vq'], 4), r['active_tokens'])
"
```

What to watch:

- **`recon`** — reconstruction L1. The headline number, but note that below roughly
  5 bits per sample it stops predicting how useful the tokenizer is downstream, so do
  not optimise it past the point where the downstream task stops improving.
- **`active_tokens` / `dead_rate`** — how much of each codebook is in use. A codebook
  collapsing to a handful of entries is the classic VQ failure, and it shows up here
  long before it shows up in the loss.
- **`perplexity`** — the effective number of codebook entries used, per layer; a
  smoother version of the same signal.

## 4. Encode the corpus into tokens

The transformer trains on token sequences, not on signal, so the corpus has to be run
through the finished tokenizer once:

```bash
brieno train encode -m runs/my-tokenizer/tokenizer_best.pt \
                    -i /data/corpus/Ecoli_Native_R9_PRJEB51164.npy \
                    -o /data/tokens/Ecoli_Native_R9_PRJEB51164.npy
```

`(N, 1, chunk_size)` float16 signal becomes `(N, num_quantizers, seq_len)` uint16 token
ids, written through a memory-mapped output so peak memory stays flat. The tokenizer
checkpoint carries its own `model_config`, so there is no config to pass and no way for
the architecture to disagree with the weights.

Keep the dataset filenames: they are the keys in the transformer config, exactly as
they were in the tokenizer config.

!!! note

    A tokenizer whose codebook k-means never completed is refused rather than used — its
    codebook is still random, and encoding with it would produce meaningless tokens that
    nothing downstream would flag.

Loop over the corpus the same way as for the merge step:

```bash
for f in /data/corpus/*.npy; do
    brieno train encode -m runs/my-tokenizer/tokenizer_best.pt \
                        -i "$f" -o "/data/tokens/$(basename "$f")"
done
```

Next: [pretrain the transformer](transformer.md).
