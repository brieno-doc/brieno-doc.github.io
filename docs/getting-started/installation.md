# Installation

Brieno requires **Python 3.10 or later**.

## Install from source

```bash
git clone https://github.com/guangzhaocs/Brieno.git
cd Brieno
pip install -e ".[fast5,pod5]"
```

Check the installation:

```bash
brieno --version
```

## Optional extras

Brieno's core dependencies are `numpy`, `torch` and `tqdm`. Everything else is an
optional extra, so you only install what your data and hardware need.

| Extra    | Installs              | When you need it |
|----------|-----------------------|------------------|
| `fast5`  | `ont-fast5-api`       | Your raw signal is in FAST5 files. |
| `pod5`   | `pod5`                | Your raw signal is in POD5 files. |
| `decode` | `fast-ctc-decode`     | CRF beam search on CPU (x86_64 and aarch64 wheels). |
| `cuda`   | `cupy`                | CRF loss / Viterbi on any CUDA platform. |
| `dev`    | `pytest`              | Running the test suite. |

`fast5` and `pod5` are separate extras because most people have data in one format
only — install the one your data is in, or both:

```bash
pip install -e ".[pod5]"          # POD5 only
pip install -e ".[fast5,pod5]"    # both formats
```

## GPU requirements

!!! note "Training needs a CUDA GPU"

    Training the tokenizer and the transformer needs a CUDA GPU.

    The CRF parts of the downstream tasks additionally need a CUDA backend:
    either `ont-koi` (x86_64 wheels only) or `cupy` (builds the same kernels at run
    time on any CUDA platform, installed with the `cuda` extra). Neither is required
    for the tokenizer.
