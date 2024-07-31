# _Scaling Inference Compute with Repeated Sampling_: SWE-bench Lite Logs

In our paper [Large Language Monkeys: Scaling Inference Compute with Repeated Sampling](https://scalyresearch.stanford.edu/pubs/large_language_monkeys/), we showed [moatless-tools](https://github.com/aorwall/moatless-tools/tree/a1017b78e3e69e7d205b1a3faa83a7d19fce3fa6) and [DeepSeek-Coder-v2-Instruct](https://arxiv.org/abs/2406.11931) can resolve 56% of the problems in [SWE-bench lite](https://swebench.com). We achieved this score by independently sampling 250 candidate solutions per problem and picking correct solutions using unit tests.


This repositories contains trajectories of and evaluation logs for the `300 problems * 250 samples/problem = 75,000` samples we drew during this project. As we have \~2 orders of magnitude more data than standard runs on SWE-bench Lite, we use a [directory structure](#directory-structure) different from the one [SWE-bench uses in their experiments repo](https://github.com/swe-bench/experiments).

Further, we observed [flakiness](#flaky-tests) in some of the unit tests the [SWE-bench evaluator](https://github.com/princeton-nlp/SWE-bench) uses to grade solutions. Because of this, we also report results on a subset of SWE-bench Lite that excludes problems that have flaky tests.


## Directory Structure
```
.
├── trajectories/
│   └── <instance_id>/
│       ├── 0.json
│       ├── 1.json
│       └── ...
├── logs/
│   ├── instances_with_flaky_tests/
│   │   └── <instance_id>/
│   │       ├── <patch_hash>.run_0.eval.log
│   │       ├── <patch_hash>.run_1.eval.log
│   │       └── ...
│   └── instances_without_flaky_tests/
│       └── <instance_id>/
│           └── <patch_hash>.eval.log
├── summary.json
├── passing_instances.jsonl
└── patch_hash_to_patch.json
```

### `trajectories/`: trajectories outputted by Moatless-tools
This folder contains the trajectories of samples, grouped by problem instance id.

Some notes:
   - Each `<instance_id>` folder contains numbered JSON files (`0.json`, `1.json`, ...) containing the trajectories from the individual samples for that problem. These content of these files is the trajectory outputted by Moatless-tools. 
   - Most instances have 250 trajectories; however, for a small number of instances, a few of the samples crashed. We omit these trajectories and mark the samples as incorrect.
   - In the "finished" state, the model is listed as `gpt-4o` instead of `deepseek-coder`. This seems to be a bug in Moatless-tools: in their [runs on Anthropic's Sonnet 3.5, they still list the model as `gpt-4o`](https://github.com/swe-bench/experiments/blob/365dcc7d107f95cfe38ec9f2178a828087d9c58f/evaluation/lite/20240623_moatless_claude35sonnet/trajs/astropy__astropy-12907.json#L395). As we use Moatless-tools out of the box, with no functional modifications, we leave this unmodified.

### `logs/`: logs from running the unit tests to evaluate each patch

To evaluate the correctness of the patches outputted by Moatless-tools and Deepseek, we used the [SWE-bench evaluator](https://github.com/princeton-nlp/SWE-bench). 
This directory contains the logs from running the evaluator on the generated patch.  It's organized into two subdirectories:

#### `instances_without_flaky_tests`
This directory contains runs for problem instances where we did not observe flakiness in the tests. We include logs from one unit test run per-patch, organized by problem instance id.
#### `instances_with_flaky_tests`:
This directory contains runs for problem instances where we observe flakiness in the tests (e.g, non-deterministically, correct patches are be marked as incorrect, or incorrect patches are marked as correct). We include logs for 11 unit tests run per-patch.

### `summary.json`: summarizes our samples
   ```json
   {
     "instances_without_flaky_tests": {
       "<instance_id>": {
         "0": {
           "resolved": true,
           "patch_hash": "a2e29..."
         },
         "1": {
           "resolved": false,
           "patch_hash": "b3f40..."
         },
         ...
       },
       ...
     },
     "instances_with_flaky_tests": {
       "<instance_id>": {
         "0": {
           "test_runs": [false, false, true, true, false, ....],
           "resolved": false
           "patch_hash": "c4g51..."
         },
         ...
       },
       ...
     }
   }
   ```
This file contains the data that indicates if each sample was correct or incorrect. We group samples by problem instance ids, and problem instance ids by if the problem has flaky tests or not.

For samples of problems with flaky tests, we determine if the sample is correct ("resolved") by running 11 tests and having them vote. The `test_runs` fields contains the results from these runs: elements of this array are booleans indicating a pass or a fail

"patch_hash" is the hash of the patch the that was sampled: the file `patch_hash_to_patch.json` can be used to recover the original patch.


### `passing_instances.jsonl`: one patch/instance we patch

This file has 168 lines - one per instance we produced a correct patch for. For each problem, we picked a correct patch arbitrarily and included it in this file.

Each line is a JSON object with the following structure:
   ```json
   {
     "model_name_or_path": "moatless-tools-deepseek-completion-5",
     "model_patch": "git diff ....",
     "instance_id": "matplotlib__matplotlib-25433"
   }
   ```

To verify these patches are correct, install the [SWE-bench evaluator](https://github.com/princeton-nlp/SWE-bench) and run 
```
python3 -m swebench.harness.run_evaluation --predictions_path passing_instances.jsonl --run_id validate-correct-samples
````

Note that due to flaky tests, some of instances with flaky tests may be marked as incorrect.


### `patch_hash_to_patch.json`: a map from sha256 hashes of patches to the content of the patch
   ```json
   {
     "<patch_hash>": "<patch_content>",
     ...
   }
   ```

## Flaky tests

During our research, we observed flakiness in the unit tests that are used by SWE-bench to grade solutions.

For 30 of the 300 problem instances, the SWE-bench evaluator non-deterministically marks the golden (correct solution provided by the dataset curators) as incorrect. We identified an addition 4 problem instances in which unit tests exhibit non-determinism.

After discussion with the SWE-bench authors, we decided to report results on the full SWE-bench Lite, as well a subset of 266 problems that excludes the problems with flaky tests. We used the data in the [SWE-bench experiments](https://github.com/swe-bench/experiments) to compute the scores of other systems/models on this subset.

![image](https://github.com/user-attachments/assets/3a7a637d-c957-4414-8f83-573aac52a11e)

