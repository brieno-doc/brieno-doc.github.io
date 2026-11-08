# Transformer

## 5. Pretrain the transformer

Spans of token positions are hidden and the model predicts them from their context.
This is the stage that turns a per-chunk code into a contextual representation, and it
is what the downstream heads attach to.

=== "One GPU"

    ```bash
    brieno train transformer -c configs/transformer_dna_r9.json
    ```

=== "Four GPUs on one node"

    ```bash
    torchrun --nproc_per_node=4 -m brieno.cli train transformer -c configs/transformer_dna_r9.json
    ```

### The config

Start from
[`configs/transformer_dna_r9.json`](https://github.com/guangzhaocs/Brieno/blob/main/configs/transformer_dna_r9.json),
the recipe of the released R9.4.1 transformer. `datasets` works as it does for the
[tokenizer](tokenizer.md#the-config), except `root_dir` points at the tokenized corpus
from [step 4](tokenizer.md#4-encode-the-corpus-into-tokens).

**`masking`** — `mask_ratio` is the fraction of positions hidden and `min_span` the
length of each hidden run. Spans rather than single positions matter here: neighbouring
signal frames are highly correlated, so a model asked to fill in one isolated position
can interpolate from its neighbours without learning anything. The start probability is
corrected for span overlap, so the realised fraction matches `mask_ratio` instead of
overshooting it by `min_span`.

The same positions are masked in **every** quantizer layer at once. Masking a position
in one layer only would leave the model the same information through another codebook,
and it would learn to copy rather than to predict.

`layer_weights` sets each quantizer layer's share of the loss. The released recipe uses
`[1.0, 0.5, 0.2, 0.1]`: the first codebook carries most of the signal, and later
residual layers are progressively closer to noise, so weighting them equally would
spend most of the model's capacity predicting what cannot be predicted.

**`training`** — `epoch_length` is optimizer steps per GPU per epoch, so an epoch means
the same amount of work however many GPUs are used. `grad_accum_steps` multiplies the
effective batch without the memory of a larger one; the effective global batch is
`batch_size × world_size × grad_accum_steps`, printed at startup so it is not left to
arithmetic.

`betas` and `eps` default to `(0.9, 0.98)` and `1e-6`, the transformer convention rather
than PyTorch's AdamW defaults, and the values the released weights were trained with.
Biases and LayerNorm gains are automatically excluded from weight decay.

### Output

```
<save_dir>/<run name>/
    config.json            the config this run actually used
    history.jsonl          per-step and per-epoch metrics
    transformer_ep<N>.pt   checkpoint after each epoch
    transformer_best.pt    lowest validation loss so far
```

`--resume <checkpoint>` continues a run that hit a wall-clock limit.

### Monitoring

What to watch in `history.jsonl`:

- **`loss`** — masked-token cross-entropy, weighted across layers.
- **`top1_L1` … `top1_L4`** — per-layer accuracy at masked positions. L1 should climb
  well above the others; later layers staying near chance is expected, not a fault,
  and is why their loss weights are small.

Next: [fine-tuning](fine-tuning.md).
