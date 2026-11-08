# Data preparation

Turn raw FAST5/POD5 reads into the per-dataset signal arrays the tokenizer trains on.

## 1. Chunk the raw signal

```bash
brieno train prep -i /data/run1/pod5 -o /data/chunks/Ecoli_Native_R9 -j 8
```

Per read, this does three things:

1. **Normalise** the whole read by median / MAD. Nanopore current is measured in
   picoamperes and drifts between pores and runs; the median/MAD transform puts every
   read on the same scale without assuming a particular current level.
2. **Trim** the open-pore and stall region at the read start, using bonito's window
   scan. The threshold depends on the chemistry, so set `--modality` to match your
   data (`dna_r9`, `dna_r10`, `rna_002`, `rna_004`).
3. **Cut** the rest into overlapping fixed-length windows, written as `float16` shards
   of shape `(N, 1, chunk_size)`.

Output is one or more `<input stem>_partN.npy` files, at most
`--max-chunks-per-file` chunks each.

Useful flags: `-j/--workers` processes whole files in parallel (they are independent,
so this scales linearly with cores until the filesystem saturates); `--resume` skips
inputs that already have output; `-r/--recursive` walks subdirectories.

### Why this is not what the task commands use

`prep` is for **training data**. The task commands (`basecaller`, `demux`, …) chunk
reads in memory with a *different* cut and never read these files. Two reasons they
must differ:

- `prep` **trims** the read start; inference must not, because the trimmed samples
  carry bases that belong in the output sequence.
- `prep` **drops** reads shorter than one chunk and discards the leading remainder;
  inference pads short reads and covers every sample, because a read that produced no
  output at all is a read missing from your results.

!!! warning

    Do not feed `prep` output to a task command, and do not change `prep` to match one.

## 2. Merge the shards into one file per dataset

The tokenizer trainer reads one array per dataset, not per input file:

```
<pretrain_root>/
    Ecoli_Native_R9_PRJEB51164.npy        (N, 1, 16384) float16
    Hsapiens_Native_R9_HG001Prometh.npy
    ...
```

because the sampler needs a single index space per dataset to draw from, and a stable
train/validation split across runs.

```bash
brieno train merge -i /data/chunks/Ecoli_Native_R9 \
                   -o /data/corpus/Ecoli_Native_R9_PRJEB51164.npy
```

The merge is two passes over memory-mapped files — measure, then fill — so the corpus
never has to fit in RAM. All shards must agree on dtype and chunk shape; a mismatch
fails before anything is written and names the offending file. An existing output is
not overwritten unless you pass `--force`.

!!! tip "The output filename is the dataset name"

    That name is the key you will use in the training config. Name it after the
    dataset, not after the run it came from.

For a whole corpus, loop in the shell rather than submitting one job per dataset:

```bash
for d in /data/chunks/*/; do
    name=$(basename "$d")
    brieno train merge -i "$d" -o "/data/corpus/${name}.npy"
done
```

Next: [pretrain the tokenizer](tokenizer.md).
