# Evaluation

## Overview

This folder contains code for experimenting with our SAFE autorater and aggregating evaluation results using our proposed F1@K metric:
- Calculating F1@K at a response-level using SAFE outputs (implemented in `metric_utils.py`).
- Evaluating against FActScore data (implemented in `correlation_vs_factscore.py`).
- Running on prompt-response pairs generated by our main experimentation pipeline (implemented in `run_eval.py`).

## Measuring long-form factuality using F1@K

We proposed to extend F1 score to the long-form factuality domain by using a hyperparameter *K* to estimate the number of supported facts for which a user cares about recall up to the *K*th supported fact.
In other words, *K* is the number of supported facts required for a response to achieve full recall.
We thus calculate F1@K as follows.

Given a model response *y* that contains *S(y)* supported facts and *N(y)* not-supported facts,

```
Precision(y) = S(y) / (S(y) + N(Y))
Recall(y) = min(S(y) / K, 1)
F1@K(y) = 2 * Precision(y) * Recall(y) / (Precision(y) + Recall(y))
```

Note that if *S(y)* is zero, we simply return F1@K = 0.
Additionally, F1@K assumes that responses do not contain repeated facts.
Our full implementation can be found in `metric_utils.py`.

## Evaluating SAFE against FActScore data

### Experiment design

[FActScore](https://arxiv.org/abs/2305.14251) contains human annotations for roughly 500 prompt-response pairs, which are stored in `third_party/factscore/labeled_data`.
The human annotation data for each prompt-response pair includes (1) the list of individual facts in the response and (2) the rating of either `'Supported'`, `'Irrelevant'`, or `'Not Supported'` for each individual fact.
Based on this available data, we set up three experiments as follows.

**1. SAFE correlation with respect to human labels for splitting responses into individual facts**

The first experiment isolates the step of splitting a response into a list of individual facts contained in the response.
This experiment runs step 1 of SAFE (see the README in the `safe/` folder for a list of all steps) to split a response into a list of individual facts.
The returned results are Pearson and Spearman correlations at a prompt-response pair level between the number of individual facts from SAFE and the number of individual facts from human annotations.
With our default SAFE configuration, the expected correlations from running on all prompt-response pairs is approximately `Pearson = 0.8` and `Spearman = 0.85`.

**2. SAFE correlation with respect to human labels for rating individual facts**

The second experiment isolates the step of rating a given individual fact that has already been extracted from a prompt-response pair.
This experiment runs steps 2 through 4 of SAFE (see the README in the `safe/` folder for a list of all steps) to rate an individual fact from a prompt-response pair.
The returned results are Pearson and Spearman correlations at a prompt-response pair level between the number of facts per label from SAFE and the number of facts per label from human annotations, for each label of `'Supported'`, `'Irrelevant'`, and `'Not Supported'`.
With our default SAFE configuration, the expected correlations from running on all prompt-response pairs is approximately `Pearson = 0.9` and `Spearman = 0.9` for `'Supported'`, `Pearson = 0.3` and `Spearman = 0.0` for `'Irrelevant'`, and `Pearson = 0.63` and `Spearman = 0.65` for `'Not Supported'`.

**3. SAFE entire pipeline correlation with respect to human labels**

The third experiment combines all steps in SAFE (see the README in the `safe/` folder for a list of all steps) to both split a response into individual facts and rate all individual facts in the response.
The returned results are Pearson and Spearman correlations at a prompt-response pair level between (1) the number of individual facts from SAFE and the number of individual facts from human annotations and (2) the number of facts per label from SAFE and the number of facts per label from human annotations, for each label of `'Supported'`, `'Irrelevant'`, and `'Not Supported'`.

### Usage

To run the pipeline for evaluating SAFE against FActScore human annotations, use

```bash
python -m eval.correlation_vs_factscore
```

**(Optional)** You can also specify the following arguments:
- `--samples`: The number of prompt-response pairs to run on. If specified as a positive integer, guarantees the code to run on a random subset of the available prompt-response pairs from FActScore. `-1` by default.
- `--eval_in_parallel`: Whether to evaluate prompt-response pairs in parallel. Set to `False` for easier debugging. `True` by default.
- `--save_results`: Whether to save the output results as a JSON file. `True` by default.
- `--eval_identify_atomic_facts_ablated`: Whether to independently run Experiment 1 - SAFE correlation with respect to human labels for splitting responses into individual facts. `False` by default.
- `--eval_rate_atomic_facts_ablated`: Whether to independently run Experiment 2 - SAFE correlation with respect to human labels for rating individual facts. `False` by default.
- `--eval_entire_safe`: Whether to independently run Experiment 3 - SAFE entire pipeline correlation with respect to human labels. `False` by default.

One of either `--eval_identify_atomic_facts_ablated`, `--eval_rate_atomic_facts_ablated`, or `--eval_entire_safe` should be `True`, otherwise no experiments will be run.

## Running SAFE on prompt-response pairs

### Overview

Getting prompt-response pairs using our main experimentation pipeline in the `main/` folder will result in the results being saved as a `.json` file.
To evaluate these prompt-response pairs, `run_eval.py` includes code for looping through each non-annotated prompt-response pair and calling the SAFE pipeline to annotate the prompt-response pair with the relevant metrics.

### Usage

To run the pipeline for evaluating prompt-response pairs from our main experimentation pipeline using SAFE, use the following command, making sure to add the path to the `.json` file containing the prompt-response pairs to be evaluated to the `--result_path` argument.

```bash
python -m eval.run_eval \
    --result_path=
```

**(Optional)** You can also specify the following arguments:
- `--eval_side1`: Whether to evaluate side 1. Our main experimentation pipeline allows for side-by-side comparison, and it may not be desired to evaluate both sides. To skip evaluating side 1, set this flag to `False`. Note that if side 1 was either `'placeholder'` or `'none'`, it will be skipped even if this flag is set to `True`. `True` by default.
- `--eval_side2`: Whether to evaluate side 2. Our main experimentation pipeline allows for side-by-side comparison, and it may not be desired to evaluate both sides. To skip evaluating side 2, set this flag to `False`. Note that if side 2 was either `'placeholder'` or `'none'`, it will be skipped even if this flag is set to `True`. `True` by default.
- `--parallelize`: Whether to parallelize evaluation to speed up runtime. Setting to `False` may be useful for debugging but significantly increases the runtime of evaluation. `True` by default.
- `--max_claim`: The hyperparameter *K* to use when calculating F1@K. If set to a number that is not positive, uses `metric_utils.DEFAULT_MAX_CLAIMS`. `-1` by default.

## Unit Tests

Each file in this directory has a corresponding unit test with the `_test` suffix (e.g., `file.py` would have `file_test.py` for unit tests). Run commands for individual tests are shown in the unit test files. To run all unit tests in this directory, use

```bash
python -m unittest discover -s eval/ -p "*_test.py"
```
