# InterFuse: Scalable Detection of Memory Safety Violations via Differential Reasoning and Hierarchical Context Fusion

InterFuse is a vulnerability-detection framework for C/C++ that combines lightweight cross-function control-flow analysis, genetic-algorithm-based path sampling, risk-aware path encoding, and hierarchical context fusion.

This repository accompanies the ACM TOSEM paper:

> **InterFuse: Scalable Detection of Memory Safety Violations via Differential Reasoning and Hierarchical Context Fusion**  
> Yao Zhang, Ruijie Cai, Xiaokang Yin, Zhenjie Xie, and Shengli Liu  
> *ACM Transactions on Software Engineering and Methodology (TOSEM)*, 2026.

## Overview

InterFuse is designed for path-dependent memory-safety and memory-related vulnerabilities whose evidence may span multiple functions.

The main pipeline consists of:

1. **Lightweight cross-function control-flow construction** using Tree-sitter-based static analysis.
2. **Genetic-algorithm-based path sampling** to retain a compact set of informative execution paths.
3. **Risk-aware path encoding** for danger operations, pointer operations, and defensive checks.
4. **Hierarchical context fusion** that combines path-level evidence with global source-code context.
5. **Binary vulnerability classification** with a validation-selected decision threshold.

The released code and preprocessed path files can be used directly for training and evaluation.

## Repository Structure

A typical release is organized as follows:

```text
InterFuse/
├── src/
│   ├── run.py
│   ├── model.py
│   ├── c_cfg.py
│   ├── data_process_single.py
│   └── ...
├── parserTool/
│   ├── build.sh
│   └── ...
├── dataset/
│   ├── d2a/
│   ├── mvd/
│   ├── juliet/
│   └── ...
├── output/
└── README.md
```

The main components are:

- `src/run.py`: model training, validation, testing, threshold selection, prediction output, and error analysis.
- `src/model.py`: InterFuse neural architecture.
- `src/c_cfg.py`: CFG construction and path-generation utilities.
- `src/data_process_single.py`: preprocessing and cross-function path-data generation.
- `parserTool/`: Tree-sitter parser utilities.
- `dataset/`: JSONL benchmark files and synchronized preprocessed PKL path files.
- `output/`: trained checkpoints, logs, predictions, and evaluation outputs.

If your extracted package places the Python files at the repository root rather than under `src/`, simply remove the `src/` prefix from the commands below.

## Environment

The reference experimental environment reported in the paper is:

| Component | Reference configuration    |
| --------- | -------------------------- |
| OS        | Ubuntu 22.04 LTS           |
| GPU       | NVIDIA RTX 4090, 24 GB     |
| CPU       | Intel Xeon Gold 6248R      |
| Python    | 3.10                       |
| PyTorch   | 2.1.0                      |
| CUDA      | 12.1                       |
| Backbone  | `microsoft/unixcoder-base` |
| Parser    | Tree-sitter                |

The code also depends on common Python packages including:

- `transformers`
- `numpy`
- `scikit-learn`
- `networkx`
- `tree-sitter`
- `tqdm`

A simple environment setup is:

```bash
conda create -n interfuse python=3.10 -y
conda activate interfuse

python -m pip install --upgrade pip
pip install torch transformers numpy scikit-learn networkx tree-sitter tqdm
```

For GPU execution, install the PyTorch build appropriate for your local CUDA environment.

## Tree-sitter Parser

The released preprocessing code uses Tree-sitter.

If the pre-built parser library is incompatible with your operating system, rebuild it before preprocessing:

```bash
cd parserTool
bash build.sh
cd ..
```

This step is only required when regenerating program-analysis artifacts from source code. It is not necessary when using the released synchronized JSONL and PKL files directly.

## Data Preparation

For reproducibility, the recommended path is to use the **released JSONL files together with their matching PKL path files**.

InterFuse does not require sampled paths to be embedded directly inside each JSONL record. The model loader synchronizes the JSONL samples and preprocessed path information by `idx`.

### JSONL format

Each JSONL sample should contain at least:

```json
{
  "idx": 123,
  "func": "source code of the target function",
  "target": 1,
  "vul_type": "buffer_overflow"
}
```

Fields:

- `idx`: unique sample identifier.
- `func`: source code used as the global code stream.
- `target`: binary label, where `0` is safe and `1` is vulnerable.
- `vul_type`: vulnerability category used for per-type evaluation; this field may be omitted when unavailable.

### PKL format

The synchronized PKL file stores path information keyed by sample identifier.

The model expects each matched sample entry to contain:

```text
path_token
```

where `path_token` is a list of sampled path-code strings.

Conceptually:

```python
{
    123: {
        "path_token": [
            "path code sequence 1",
            "path code sequence 2",
            "...",
        ]
    }
}
```

`run.py` accepts either integer or string forms of the same `idx`.

By default, at most `15` paths are consumed for each sample.

### Example directory layout

```text
dataset/
└── d2a/
    ├── train_cdata_synced.jsonl
    ├── valid_cdata_synced.jsonl
    ├── test_cdata_merged.jsonl
    ├── short_3path_cdata_nobalance_train.pkl
    ├── short_3path_cdata_nobalance_valid.pkl
    └── short_3path_cdata_nobalance_test.pkl
```

The filenames above correspond to the default argument names in `run.py`. If your released files use different names, pass them explicitly with the corresponding command-line options.

## D2A Reference Training Configuration

The final D2A configuration reported in the paper is:

| Parameter                |                      Value |
| ------------------------ | -------------------------: |
| Base encoder             | `microsoft/unixcoder-base` |
| Learning rate            |                     `1e-5` |
| Warm-up ratio            |                     `0.06` |
| Weight decay             |                     `0.01` |
| Per-device batch size    |                        `1` |
| Gradient accumulation    |                       `12` |
| Maximum epochs           |                       `25` |
| Early-stopping patience  |                        `5` |
| Maximum sequence length  |                      `860` |
| Maximum paths per sample |                       `15` |
| Path-dropout rate        |                      `0.2` |
| GA population size       |                       `25` |
| GA generations           |                       `10` |
| GA mutation rate         |                     `0.25` |

The paper's interprocedural depth setting corresponds to contexts spanning up to four functions, i.e., at most three call transitions along a retained path.

## Training

The following command reproduces the main D2A training configuration using the current `run.py` interface:

```bash
python src/run.py \
  --output_dir ./output/d2a \
  --data_dir ./dataset/d2a \
  --pkl_dir ./dataset/d2a \
  --model_name_or_path microsoft/unixcoder-base \
  --do_train \
  --do_eval \
  --train_file_name train_cdata_synced.jsonl \
  --valid_file_name valid_cdata_synced.jsonl \
  --pkl_file_name_train short_3path_cdata_nobalance_train.pkl \
  --pkl_file_name_valid short_3path_cdata_nobalance_valid.pkl \
  --block_size 860 \
  --max_paths 15 \
  --train_batch_size 1 \
  --eval_batch_size 1 \
  --gradient_accumulation_steps 12 \
  --learning_rate 1e-5 \
  --weight_decay 0.01 \
  --num_train_epochs 25 \
  --warmup_ratio 0.06 \
  --path_dropout_rate 0.2 \
  --early_stopping_patience 5 \
  --metric_for_best_model f1 \
  --seed 42
```

During training, the validation split is used to select the best checkpoint and classification threshold.

The best checkpoint is saved under:

```text
output/d2a/checkpoint-best/
```

Typical files include:

```text
checkpoint-best/
├── model.bin
├── best_params.json
├── training_args.bin
├── config.json
└── tokenizer files
```

The latest resumable training state is stored separately under:

```text
output/d2a/checkpoint-latest/checkpoint.bin
```

## Evaluation with a Released Checkpoint

To evaluate a released checkpoint, place the checkpoint files under:

```text
output/d2a/checkpoint-best/
```

At minimum, `run.py` requires:

```text
output/d2a/checkpoint-best/model.bin
```

For the validation-selected threshold used during the experiment, also keep:

```text
output/d2a/checkpoint-best/best_params.json
```

Then run:

```bash
python src/run.py \
  --output_dir ./output/d2a \
  --data_dir ./dataset/d2a \
  --pkl_dir ./dataset/d2a \
  --model_name_or_path microsoft/unixcoder-base \
  --do_test \
  --test_file_name test_cdata_merged.jsonl \
  --pkl_file_name_test short_3path_cdata_nobalance_test.pkl \
  --block_size 860 \
  --max_paths 15 \
  --eval_batch_size 1 \
  --seed 42
```

If `best_params.json` is available, the stored validation-selected threshold is loaded automatically. Otherwise, the testing code falls back to a threshold of `0.5`.

## Evaluation Outputs

Testing writes the main log to:

```text
output/d2a/run_log_v5.log
```

Overall predictions are written to:

```text
output/d2a/predictions_overall.txt
```

The testing stage reports:

- Accuracy
- Recall
- Precision
- F1
- MCC
- PR-AUC
- ROC-AUC

For D2A-style per-type evaluation, the code also evaluates each detected vulnerability type against safe samples.

Error-analysis files are written to the output directory, for example:

```text
error_analysis_fp_Overall.jsonl
error_analysis_fn_Overall.jsonl
```

## Using a Balanced Per-Type Test Set

For vulnerability-type-specific evaluation with an equal number of safe and vulnerable samples, add:

```bash
--balance_test_set
```

For example:

```bash
python src/run.py \
  --output_dir ./output/d2a \
  --data_dir ./dataset/d2a \
  --pkl_dir ./dataset/d2a \
  --model_name_or_path microsoft/unixcoder-base \
  --do_test \
  --balance_test_set \
  --test_file_name test_cdata_merged.jsonl \
  --pkl_file_name_test short_3path_cdata_nobalance_test.pkl \
  --block_size 860 \
  --max_paths 15 \
  --eval_batch_size 1 \
  --seed 42
```

## Caching and Runtime Options

Useful optional arguments include:

```text
--use_cache
--fp16
--num_preprocessing_workers
--num_dataloader_workers
--preprocessing_chunksize
```

Example:

```bash
python src/run.py \
  --output_dir ./output/d2a \
  --data_dir ./dataset/d2a \
  --pkl_dir ./dataset/d2a \
  --model_name_or_path microsoft/unixcoder-base \
  --do_test \
  --use_cache \
  --fp16
```

Use `--fp16` only on hardware that supports mixed-precision execution reliably.

## Preprocessing from Source Code

For exact reproduction of the reported model results, we recommend using the released preprocessed PKL files.

If you want to regenerate path files from source code, the preprocessing pipeline is implemented primarily in:

```text
src/data_process_single.py
src/c_cfg.py
parserTool/
```

Before running preprocessing, ensure that the Tree-sitter parser is available.

To inspect the arguments supported by the released preprocessing script:

```bash
python src/data_process_single.py --help
```

The preprocessing stage generates path data that are subsequently loaded by `run.py` through the synchronized PKL files.

Because newly generated path sets can depend on preprocessing inputs, parser versions, and random seeds, keep the released JSONL/PKL pairs together when reproducing the reported checkpoint results.

## Benchmarks

The paper evaluates InterFuse on D2A, MVD, Juliet, and ReposVul.

### D2A

The synchronized D2A test split contains:

| Class      |   Samples |
| ---------- | --------: |
| Vulnerable |     1,339 |
| Safe       |     1,331 |
| **Total**  | **2,670** |

### MVD

After filtering to retain interprocedural real-world samples, MVD contains:

| Split      |   Samples |
| ---------- | --------: |
| Train      |     5,273 |
| Validation |       674 |
| Test       |       673 |
| **Total**  | **6,620** |

### Juliet

The synchronized Juliet test split contains `3,161` samples.

### ReposVul

The transfer evaluation uses a fixed `120`-sample ReposVul test subset. A separate labeled set of `20` ReposVul samples is used only for threshold calibration; model parameters are not fine-tuned on ReposVul.

## Main Results

The final paper reports the following overall results:

| Benchmark | Accuracy | Precision | Recall |       F1 |
| --------- | -------: | --------: | -----: | -------: |
| D2A       |     84.5 |      83.6 |   85.8 | **84.7** |
| MVD       |     75.8 |      76.5 |   74.4 | **75.4** |
| Juliet    |     97.1 |      95.2 |   99.3 | **97.2** |

On the fixed ReposVul transfer test subset, InterFuse reports **64.3% F1** after threshold calibration without target-domain model fine-tuning.

The graph-construction stage reported in the paper requires approximately `1.6 s` per D2A sample in the reference environment, corresponding to an `11.7x` speedup over the evaluated SnapVuln graph-construction pipeline.

## Implementation

The executable artifact should be treated as the authoritative reference for low-level implementation details.

Direct call sites are resolved by matching AST-extracted callee identifiers to function definitions available in the same sample. Cross-function transitions from caller call sites to the corresponding callee entries are used during path generation.

The sampled path sequences are stored in the preprocessed `path_token` data and are consumed by the path stream of the model. Edge labels themselves are not direct neural-network inputs.

Risk-aware markers are used for three lightweight categories:

- danger operations,
- pointer operations,
- defensive checks.

These signals are incorporated into path encoding, while the global stream encodes the source-code context associated with the JSONL sample.

## Artifact Availability

The repository contains the source code and experiment implementation.

Large processed datasets and trained checkpoints can be distributed separately if necessary:

```text
<DATA_AND_CHECKPOINT_URL>
```

Replace the placeholder above with the permanent release location before making the repository public.

When redistributing third-party datasets or models, retain their original licenses and terms of use.

## Citation

If you use InterFuse, please cite:

```bibtex
@article{zhang2026interfuse,
  title   = {InterFuse: Scalable Detection of Memory Safety Violations via Differential Reasoning and Hierarchical Context Fusion},
  author  = {Zhang, Yao and Cai, Ruijie and Yin, Xiaokang and Xie, Zhenjie and Liu, Shengli},
  journal = {ACM Transactions on Software Engineering and Methodology},
  year    = {2026}
}
```

Add the final DOI, volume, issue, and article/page information after ACM assigns the Version of Record metadata.

## License

Please refer to the repository license for the InterFuse source code. Third-party datasets, pretrained models, and external libraries remain subject to their respective licenses.
