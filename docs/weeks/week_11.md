# Week 11 (Days 71–77): Intervals, Fine-Tuning, and Coding Agents

## This week

- **You will learn:**
    - **DSA:** how sorting plus one greedy sweep solves interval problems, and how to prove a greedy choice is safe.
    - **ML/DL:** how fine-tuning works, and how LoRA trains a small low-rank update instead of the whole model.
    - **AI agents:** how coding agents read, edit, and test a repo, and how to review their work like a maintainer.
    - **Software dev:** how to profile a request path and fix the slowest part first.
- **Goal:** by Sunday you can fine-tune a small model for one task in your product, show with numbers whether it beats the prompt-only baseline, and explain every PR an agent wrote for you.
- **Weekly build:** a LoRA fine-tuned model serving one feature of your product, behind a flag, with a fallback. See [Weekly build](#weekly-build).
- **AI rule (Days 61–100):** AI agents, you review. Agents write code. You write the spec, review the PR, run the evals, and reject bad output.

## DSA: intervals and greedy

**Read first:** the Greedy and Intervals rows of [DSA to ML](https://mldsa.github.io/docs/index-dsa-to-ml/) in *From LeetCode to Learning Systems*.

**Focus.** Interval problems look messy until you sort. After sorting by start or by end, one pass decides everything, because each interval only interacts with its neighbors. Greedy is the same idea in general: make the locally best choice and never revisit it. That only works when you can prove it. The standard proof is an **exchange argument**: take any optimal answer, swap in your greedy choice, and show the answer is no worse.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 71 | [56. Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Sort by start, extend the current end |
| 71 | [57. Insert Interval](https://leetcode.com/problems/insert-interval/) | Three phases: before, overlap, after |
| 72 | [435. Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/) | Sort by end, keep the earliest finisher |
| 72 | [452. Minimum Number of Arrows to Burst Balloons](https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/) | Same greedy, counted the other way |
| 73 | [55. Jump Game](https://leetcode.com/problems/jump-game/) | Track the farthest reachable index |
| 74 | [134. Gas Station](https://leetcode.com/problems/gas-station/) | Reset the start when the tank goes negative |
| 75 | [763. Partition Labels](https://leetcode.com/problems/partition-labels/) | Last occurrence defines each part's end |

### Depth ladder: Merge Intervals

1. **Solve.** Sort by start. Keep a current interval. If the next one starts before the current end, extend the end with `max`. Otherwise emit the current one and start a new one. O(n log n) time, O(n) for the output. Write tests for the given examples first.
2. **Vary.**
    - The input is already sorted and one new interval arrives. That is Insert Interval, and it is O(n) with no sort.
    - Count the maximum number of intervals open at once (meeting rooms). Turn each interval into a `+1` event at the start and a `-1` event at the end, sort the events, and keep a running sum. How do you order a start and an end at the same time?
    - Merge intervals that are within a gap `g` of each other, not only overlapping ones. This is sessionization: user events less than 30 minutes apart form one session.
    - Rectangles instead of intervals. Why does the 1-D sweep no longer settle everything?
3. **Break the assumptions.**
    - `[1, 2]` and `[2, 3]`. Do they merge? It depends on whether intervals are **closed** (`[start, end]`) or **half-open** (`[start, end)`). LeetCode treats them as closed. Most time ranges in software are half-open. Write down which one your code assumes and test both.
    - `start > end` in the input. Bad data. Does your code silently produce garbage, or reject it?
    - Timestamps from different time zones, or across a daylight-saving change. Two intervals that look disjoint in local time overlap in UTC.
    - Ten billion intervals on disk. Sort externally, then the merge is still one streaming pass. Why does the merge itself need only O(1) memory?
4. **ML lens.**
    - **Span merging in NER.** A token classifier predicts labels per sub-word token. The pipeline merges adjacent tokens into entity spans. Read `group_entities` and `aggregate_words` in Hugging Face `transformers`, `src/transformers/pipelines/token_classification.py`, and the `AggregationStrategy` enum.
    - **RAG context.** Retrieval often returns overlapping chunks from the same document. Merge overlapping character spans before you build the prompt. Measure the tokens you save on your Week 6 RAG.
    - **Sessionization** is a common feature-engineering step: number of sessions, mean session length.
5. **In the wild.**
    - pandas: `IntervalIndex.is_overlapping` in `pandas/core/indexes/interval.py`, backed by the interval tree in `pandas/_libs/intervaltree.pxi.in`. What does it do with `closed="left"` vs `closed="both"`?
    - Linux kernel: `include/linux/interval_tree.h`. Each node stores `start`, `last`, and `__subtree_last`. Note that `last` is *inclusive*. Why does each node store the largest end in its subtree, and what query does that make fast?

### Depth ladder: Non-overlapping Intervals

1. **Solve.** Sort by **end**. Keep the interval that finishes first, skip everything that overlaps it, repeat. The answer is `n` minus the number kept. O(n log n).
2. **Vary.**
    - Minimum Number of Arrows (452) is the same greedy. Each arrow goes at the end of the earliest-finishing balloon.
    - Each interval has a weight, and you maximize total weight. Greedy fails. Sort by end and use DP with binary search for the last compatible interval. This connects to Week 10.
    - You have `k` rooms, not one. Which greedy now works, and what data structure does it need?
3. **Break the assumptions.**
    - Sort by **start** instead and show it fails: `[[1, 100], [2, 3], [4, 5]]`. Sort by length and find a counterexample too.
    - **Prove it.** Write the exchange argument in four sentences in your log. Any optimal schedule can swap its first interval for the earliest-finishing one without losing anything.
    - Touching intervals. In 435, `[1, 2]` and `[2, 3]` do **not** overlap. In 452, balloons `[1, 2]` and `[2, 3]` share one arrow. Same data, different rule. Your comparison operator is the whole difference.
4. **ML lens.**
    - **Greedy decoding** picks the most likely next token every step. Unlike interval scheduling, there is no exchange argument, so it does not maximize the probability of the whole sequence. See greedy decoding vs sampling in the [inference chapter](https://dpaul0501.github.io/llm-agent-reads/part_06_inference_and_tricks/). Find a short example where greedy and beam search disagree.
    - **Batch scheduling.** An inference server admits requests greedily into each step under a token budget.
5. **In the wild.**
    - vLLM: `schedule()` in `vllm/v1/core/sched/scheduler.py`. Find `token_budget`. How does it decide which waiting requests to admit, and what happens to running requests when memory runs out (search for `preempt`)?

### Depth ladder: Gas Station

1. **Solve.** If total gas is less than total cost, return `-1`. Otherwise scan once with a tank. When the tank goes negative at station `i`, no start between the old start and `i` can work, so the new start is `i + 1`. O(n) time, O(1) space.
2. **Vary.**
    - Return every valid start. Under the problem's guarantee there is at most one. Remove the guarantee and what changes?
    - The tank has a capacity limit.
    - Minimize the number of refueling stops on a line (LeetCode 871). Greedy with a heap: refuel from the best station you have already passed, only when you must.
3. **Break the assumptions.**
    - Prove the reset rule. Why can you skip every station between the old start and `i`?
    - Floats. Total gas equals total cost up to rounding. Your `total < 0` check returns `-1` when the true answer exists. What tolerance do you use, and where does it go?
    - `n = 1`. An empty input.
4. **ML lens.**
    - The answer is the index just after the minimum of the prefix sum of `gas - cost`. Write it as `np.argmin(np.cumsum(gas - cost))` and compare it with your loop. The same prefix-minimum trick finds the worst drawdown in a cumulative-reward curve in an RL log.
    - **Token budgets.** An agent run spends tokens at each step and gets new budget at each checkpoint. Where would it first run out? Same scan.
5. **In the wild.**
    - `itertools.accumulate` and `np.cumsum` compute the running tank. Benchmark your loop against the NumPy version at n = 10⁶.

### Depth ladder: Insert Interval

- **Vary:** find the overlap boundaries with `bisect` (`Lib/bisect.py`) in O(log n). Why is the whole operation still O(n) on a Python list?
- **Break:** the new interval is empty, before everything, after everything, or covers everything. Write one test for each.
- **ML lens:** keeping a sorted list of time windows for feature backfills, where each new run inserts its range.
- **In the wild:** calendar apps store busy time as merged intervals. Implement "find a free 30-minute slot for three people" with your merge code.

### Depth ladder: Minimum Number of Arrows

- **Break:** the points include `-2³¹` and `2³¹ - 1`. A comparator written as `a[1] - b[1]` overflows in Java or C++. Python does not overflow. Explain why, and say what you would use in a fixed-width language.
- **ML lens:** the same greedy covers a set of time windows with the fewest checkpoints, for example the fewest eval runs that touch every training phase.

### Day 73 extra: Jump Game

- **Vary:** Jump Game II (LeetCode 45) asks for the fewest jumps. It is BFS by levels, done greedily in O(n). Compare it with a real BFS.
- **Break:** `nums = [0]`. A zero in the middle that you can jump over, and one you cannot.
- **ML lens:** reachability. "Can any path through this pipeline reach state X?" is the same scan.

### Day 75 extra: Partition Labels

- **Vary:** record the last index of each letter, then cut when the scan reaches the current part's farthest last index. This is merge intervals in disguise: each letter is an interval from its first to its last index.
- **ML lens:** **group-aware splits.** If one user's rows land in both train and test, the model memorizes the user and the test score lies. Grouping rows so that no user crosses a split is the same problem. scikit-learn's `GroupKFold` does it. You use it in this week's build and again in [Week 12](week_12.md).

## ML/DL: fine-tuning and LoRA

### Learn

- What fine-tuning changes: the same next-token loss as pretraining, on your data ([training chapter](https://dpaul0501.github.io/llm-agent-reads/part_05_training_and_optimization/), section 10)
- Full fine-tuning vs parameter-efficient fine-tuning. Memory for weights, gradients, and optimizer state
- **LoRA:** freeze `W`, learn `ΔW = B·A` with rank `r`, and scale it by `lora_alpha / r`. `B` starts at zero, so training starts from the base model exactly
- Which layers to adapt (`target_modules`), and how to count trainable parameters
- **Loss masking:** compute the loss only on answer tokens, not prompt tokens. PyTorch's `CrossEntropyLoss` skips label `-100` by default (`ignore_index=-100`)
- Overfitting on a few hundred examples, and forgetting what the base model could do
- Merging the adapter into the base weights for inference

Read the LoRA paper (Hu et al., 2021, arXiv:2106.09685) sections 4 and 7. Skim QLoRA (Dettmers et al., 2023, arXiv:2305.14314) for the idea of training adapters on a quantized base.

### Read the code

**1. Hugging Face `peft`, [`src/peft/tuners/lora/config.py`](https://github.com/huggingface/peft/blob/main/src/peft/tuners/lora/config.py) (`LoraConfig`) and [`src/peft/tuners/lora/layer.py`](https://github.com/huggingface/peft/blob/main/src/peft/tuners/lora/layer.py).** Answer in your log:

- In `update_layer`, find `self.scaling[adapter_name] = lora_alpha / r`. Double `r` and leave `lora_alpha` alone. What happens to the size of the update? Now find the `use_rslora` branch, which uses `lora_alpha / math.sqrt(r)`. Why would that be better at large `r`?
- In `reset_lora_parameters`, `lora_A` gets `kaiming_uniform_` and `lora_B` gets `zeros_`. Why does a zero `B` make the adapter a no-op before training? What would happen if `A` were zero too?
- `lora_dropout` becomes `nn.Dropout` when it is above `0.0`, and `nn.Identity` otherwise. Where in the forward pass is it applied: to the input of `A`, or to the output?
- `Linear.merge` has a `safe_merge` flag. What does it check before writing the merged weight? Find the `torch.isfinite` call.
- `get_delta_weight` casts to float32 on CPU when the weights are float16 or bfloat16. Why?

**2. Your own LoRA, in 30 lines.** Wrap one `nn.Linear` from your Week 10 GPT with `A` and `B` parameters, freeze the base weight, and train. Then check that your merged weight `W + scale · B @ A` gives the same output as the unmerged forward pass, within a tolerance you state.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `LoraConfig` | `r` | `8` | Rank of the update | `r=1` may underfit the task. Large `r` costs memory and overfits a small dataset |
| `LoraConfig` | `lora_alpha` | `8` | Numerator of the scale `lora_alpha / r` | Too large: the update swamps the base model and training diverges. Too small: nothing moves |
| `LoraConfig` | `lora_dropout` | `0.0` | Dropout on the adapter path | `0.0` on 200 examples overfits fast. High values slow learning |
| `LoraConfig` | `target_modules` | `None` | Which layers get adapters | `None` falls back to a per-architecture default when peft knows the model (check the docs). Attention-only vs all linear layers can change quality a lot |
| `LoraConfig` | `bias` | `"none"` | Whether biases train | `"all"` changes the base model's behavior even with the adapter disabled |
| `LoraConfig` | `use_rslora` | `False` | Scale by `lora_alpha / sqrt(r)` | With `False`, raising `r` shrinks the effective step size |
| `LoraConfig` | `init_lora_weights` | `True` | `B = 0` init | `False` gives random `A` and `B`, so the adapter changes outputs before any training |
| `TrainingArguments` | `learning_rate` | `5e-5` | Step size | LoRA often needs a higher rate than full fine-tuning. Too high and the loss spikes |
| `TrainingArguments` | `num_train_epochs` | `3.0` | Passes over the data | Many epochs on a few hundred rows memorizes them. Watch validation loss |
| `TrainingArguments` | `gradient_accumulation_steps` | `1` | Effective batch size | Large values with a tiny dataset mean very few optimizer steps |
| `CrossEntropyLoss` | `ignore_index` | `-100` | Labels skipped in the loss | Forget to set prompt labels to `-100` and the model learns to repeat prompts |

### Break it

- Set both `A` and `B` to zero and train for 100 steps. Show the loss never moves, and explain from the gradient why.
- Train once with the loss on prompt tokens and once with prompt tokens masked to `-100`. Compare outputs on five held-out inputs. Which one starts echoing the prompt?

## AI agents: coding agents

### Learn

- What a coding agent does: a loop that reads files, searches the repo, edits, runs commands and tests, and repeats ([agents chapter](https://mldsa.github.io/docs/ml-spine/chapter-agents/))
- How it picks context: file search, a repo map, the files you name in the spec
- **The spec is the prompt.** Goal, constraints, files in scope, files out of scope, and the acceptance tests it must pass
- The PR is the unit of review. Small PRs, one purpose each
- Sandboxing and permissions: what the agent may run, and what it may not touch
- Common failure modes:
    - It edits the test so the test passes.
    - It changes files outside the scope you set.
    - It calls a function or flag that does not exist.
    - It deletes error handling to make a stack trace go away.
    - It changes behavior that no test covers.

### Experiment

Pick 10 real tasks from your product's backlog. For each one, write a spec with acceptance tests **before** the agent starts. Let the agent open a PR. Review it yourself. Log one row per PR:

| Field | Example |
|---|---|
| Task | "Add a fallback when the fine-tuned model times out" |
| Spec length | 14 lines |
| Decision | accept / accept after edits / reject |
| Reason (if not accepted) | edited a test; out of scope; wrong API; missing test; style |
| Files touched vs files in scope | 4 vs 2 |
| Review time | 12 minutes |
| Seeded bug caught by its tests? | yes / no |

Compute your acceptance rate and your top rejection reason. Then rewrite the three worst specs and rerun them. Did the acceptance rate change? Your review log is evidence for the defense.

### Break it

- Give the agent a failing test where the bug is in the code. Does it fix the code, or weaken the test?
- Give it a vague spec: "make search faster". Count the files it touches. Compare with a spec that names the function and the latency target.
- Plant a docstring that describes the wrong behavior. Does the agent trust the docstring or the code?

### ML lens

Your accept/reject log is a labeled eval set, the same idea as Week 7's agent evals. SWE-bench grades an agent PR with two test sets: tests that must go from failing to passing, and tests that must keep passing. Your seeded-bug check plays the first role and your existing suite plays the second.

### In the wild

- SWE-bench: `swebench/harness/grading.py` in [SWE-bench/SWE-bench](https://github.com/SWE-bench/SWE-bench). Find `FAIL_TO_PASS` and `PASS_TO_PASS`. Why are both needed?
- aider: `aider/repomap.py` in [Aider-AI/aider](https://github.com/Aider-AI/aider). It parses the repo with tree-sitter, builds a graph of definitions and references, and ranks files with `nx.pagerank` to decide what code to show the model. Next week's graph algorithms are inside your coding agent.
- [SWE-agent](https://github.com/SWE-agent/SWE-agent): read how it defines the commands the agent may run.

## Software dev: profiling

### Learn

- Measure before you optimize. Wall time vs CPU time. `time.perf_counter`
- Deterministic profilers (`cProfile`) record every call. Sampling profilers ([py-spy](https://github.com/benfred/py-spy)) take snapshots and cost much less
- `pstats`: `tottime` (time in the function itself) vs `cumtime` (including calls it makes). `sort_stats`, `print_callers`
- Memory: `tracemalloc` snapshots and diffs
- Model code: `torch.profiler.profile` with `record_shapes`, `profile_memory`, `with_stack` (all default `False`)
- Report p50 and p95, not the mean. One slow request in twenty is a p95 problem
- Read the source:
    - CPython 3.13: `Lib/cProfile.py` is a thin wrapper around the C profiler in `Modules/_lsprof.c`. On CPython's main branch it is now a compatibility wrapper around a new `profiling.tracing` module, next to a new `profiling.sampling` module. Why would CPython add a sampling profiler to the standard library?
    - `Lib/pstats.py`: the `SortKey` class. What is the difference between `CUMULATIVE` and `TIME`?

### Apply it

Profile your product's full request path: retrieval, the model call (API or fine-tuned), tool calls, and post-processing.

- Run py-spy on a server under load and save a flame graph.
- Run cProfile on one request and sort by `cumtime`.
- Find the top three costs. Fix one. Have a coding agent write the fix from your spec, and review it.

### Verify

- A benchmark script in the repo that sends the same 100 requests and prints p50 and p95.
- Numbers before and after the fix, from the same script on the same machine.
- Run once with cProfile on and once off. How much did the profiler itself slow things down? If the overhead is large, your profile may point at the wrong function.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 71 | Merge Intervals, Insert Interval | Fine-tuning vs LoRA; read `LoraConfig` | How coding agents work; write the first spec | `perf_counter`, p50 and p95 |
| 72 | Non-overlapping Intervals, Arrows | Read `layer.py`: scaling, init, merge | Run 5 specs; start the review log | cProfile and pstats |
| 73 | Jump Game | Your own LoRA in 30 lines | Break it: tests, scope, docstrings | py-spy flame graph under load |
| 74 | Gas Station | Build the dataset; group-aware split | Run 5 more specs; rewrite the worst 3 | `tracemalloc`; `torch.profiler` |
| 75 | Partition Labels | Train, mask the loss, compare to baseline | Compute acceptance rate and top reason | Fix the top cost; before and after |
| 76 | **Weekly build** | | | |
| 77 | **Defend and publish** | | | |

## Weekly build

### Fine-tuned model inside your product

Pick one narrow task in your product where the prompt-only LLM call is weak, slow, or expensive. Good candidates are classification, extraction into your schema, or rewriting into your house format. Fine-tune a small open model with LoRA for that task. Ship it behind a flag and compare it against the prompt-only baseline on held-out data.

If you have no GPU, use a free notebook GPU, or apply your own LoRA to your Week 10 GPT. If you train on outputs from an LLM API, read your provider's terms first.

**Required behavior**

- **Data (DSA + ML/DL).**
    - A few hundred examples from your eval set and real (consented) user inputs.
    - Deduplicated.
    - Split by group (user or document) so that no group appears in two splits.
    - A short data card: source, size, label rules, known gaps.
- **Training (ML/DL).**
    - LoRA through `peft`, with the config in a file and the seed fixed.
    - Print trainable vs total parameters.
    - Prompt tokens masked out of the loss.
    - Train and validation loss curves saved.
    - Merged and unmerged models agree within a stated tolerance.
- **Integration (AI agents).**
    - The fine-tuned model sits behind the same interface as your LLM call.
    - A flag picks the path. On error or timeout it falls back to the API path.
- **Coding agents (AI agents).**
    - At least 5 PRs written by a coding agent from your specs, each reviewed by you.
    - The review log from the experiment, committed to the repo.
- **Profiling (Software dev).**
    - p50 and p95 latency, and cost per 1,000 requests, for both paths.
    - One fix guided by a profile, with before-and-after numbers.
- **Engineering.**
    - Tests for the split, the masking, the fallback, and merge equivalence.
    - A README section on how to retrain.

**Stretch**

- Sweep `r` over 1, 4, 16, 64, with and without `use_rslora`. Plot validation score against `r`.
- Train on a quantized base (QLoRA style). Measure memory saved and quality lost.
- Route by confidence: easy cases to the fine-tuned model, hard cases to the API.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 71 | "Do `[1, 2]` and `[2, 3]` overlap?" Closed vs half-open intervals, and the Linux kernel's inclusive `last`. |
| 72 | "Sort by start and greedy fails." The counterexample, the exchange argument, and why greedy decoding has no such proof. |
| 73 | "LoRA's `B` starts at zero." What that buys, and what happens when `A` is zero too. |
| 74 | "Doubling LoRA's rank halved my update." `lora_alpha / r` vs `lora_alpha / sqrt(r)`. |
| 75 | "I let a coding agent write 10 PRs. I rejected N." Your acceptance rate and the top reason. |
| 76 | Fine-tuned model vs prompt-only baseline: quality, p95 latency, and cost per 1,000 requests. |
| 77 | What the defense exposed, and what you fixed. |

**This week's write-up: Hackathon.** Submit an entry to an open online hackathon built around this week's fine-tuned feature. Check the rules on data and model licenses before you submit.

If you plan to file a provisional patent in [Week 13](week_13.md), keep the core idea out of this week's posts and your hackathon entry. See the [publishing and patents warning](../how-it-works.md).

## Value-creation step

!!! note "Your paths"
    This step is written for the Product path. On other paths, use this time for your next milestone in [Choose your paths](../paths.md).

Propose a paid pilot or a letter of intent to both of your paid-pilot candidates from Week 10. Send each a one-page proposal:

- The problem, in their words from your interviews.
- What you will deliver, and for how long (for example, four weeks).
- One success metric you both agree to measure.
- The price, or for a letter of intent, the price they would pay if the metric is met.

Log each response: yes, no, or not yet, and the reason in their words.
