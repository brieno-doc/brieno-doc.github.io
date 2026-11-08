# Downstream tasks

The pretrained Brieno transformer can be fine-tuned with task-specific heads. With frozen
or fine-tuned features, the model:

| Task | Result |
|------|--------|
| **Species identification** | Identifies species without requiring reference genomes at inference time. |
| **Native-barcode demultiplexing** | Higher overall accuracy than Dorado. |
| **Basecalling** | Approaches Dorado’s performance across six genomes. |
| **CpG methylation detection** | Higher read-level AUROC than Dorado’s 5mC model. |

See the [preprint](https://www.biorxiv.org/content/10.64898/2026.11.05.764536v1) for the
full benchmarks.

## Running a task model

!!! info "Coming soon"

    The task models are published as trained weights on
    [Hugging Face](https://huggingface.co/guangzhaocs/Brieno). Command-line tools for
    running them are being prepared for a later release, as are the fine-tuning scripts
    that produced them.

<!-- TODO: document `brieno basecaller`, `demux`, `classifier`, `modcaller` and
     `download` once they are available in the CLI. -->

## Training your own model

The whole pretraining pipeline is already available — chunking raw reads, pretraining
the tokenizer and the transformer. See the [training guide](../training/index.md).
