# Week 14 (Days 92–100): Mock Interviews, Polish, and Demo Day

## This week

- **You will learn:**
    - **DSA:** how to solve an unseen problem in 45 minutes while thinking aloud, then take your own solution through the depth ladder.
    - **ML/DL:** how to polish a model for launch: error analysis by slice, calibrated thresholds, one final test-set run, and a model card.
    - **AI agents:** how to find and fix the failures real users hit, and how to keep the agent reliable on launch day.
    - **Software dev:** how to run a launch checklist where every item has evidence.
- **Goal:** on Day 100 you demo your capstone live, launch it publicly, and present your users, revenue, and lessons.
- **Weekly build:** the launched capstone, a demo-day presentation, and a demo video. See [Weekly build](#weekly-build).
- **AI rule (Days 61–100):** AI agents, you review. Agents write code. You write the spec, review the PR, run the evals, and reject bad output. In the mock interviews, no AI at all.

This week has nine days. Days 92–98 polish, Day 99 is the dress rehearsal, and Day 100 is demo day and launch.

## DSA: mock interviews

**Focus.** For 13 weeks you solved problems knowing the pattern in advance. Interviews and real work do not tell you the pattern. These 8 problems are mixed across patterns from earlier weeks, and they bring your total to 100. Do each one as a mock interview first, then run the depth ladder on your own solution.

### Mock-interview protocol

Pair with another participant. One interviews, one solves, then swap. Record the session if both agree.

| Minutes | What the candidate does |
|---|---|
| 0–5 | Restate the problem. Ask about input size, edge cases, and what to return. Write two examples. |
| 5–10 | Say the brute force and its complexity. Then name the pattern and the better approach. Get a nod before coding. |
| 10–35 | Code while thinking aloud. Say what each block does and why. |
| 35–45 | Test by hand: the examples, then an edge case you choose. State the final time and space complexity. |

**Rules.** No AI, no autocomplete, no searching. The interviewer gives at most two hints and writes down when.

**The interviewer scores** four things, 1–4 each: clarifying questions, choice of approach, code correctness, and testing. Write one sentence of feedback for each.

**After the mock**, on your own time, take your solution through the [depth ladder](../how-it-works.md#the-depth-ladder): write tests, vary the problem, break an assumption, find the ML lens, and read one real source file. Hints for each problem are below.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 92 | [560. Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/) | Hashing, prefix sums: count earlier prefix sums equal to `prefix − k` |
| 93 | [1011. Capacity To Ship Packages Within D Days](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) | Binary search on the answer: is capacity c enough? |
| 94 | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Hashing, design: dict of index plus array; swap with the last to delete |
| 95 | [437. Path Sum III](https://leetcode.com/problems/path-sum-iii/) | Trees: prefix sums along the root-to-node path |
| 96 | [502. IPO](https://leetcode.com/problems/ipo/) | Heaps, greedy: unlock by capital, take the best profit |
| 96 | [909. Snakes and Ladders](https://leetcode.com/problems/snakes-and-ladders/) | Graphs, BFS: shortest path over squares; convert numbers to cells |
| 97 | [1235. Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling/) | DP, intervals: sort by end; DP with binary search |
| 98 | [907. Sum of Subarray Minimums](https://leetcode.com/problems/sum-of-subarray-minimums/) | Monotonic stack: count how many subarrays each element is the minimum of |

### After the mock: ladder hints

- **Subarray Sum Equals K.**
    - **Break:** with negative numbers, a sliding window fails. Show the case. Initialize the count map with `{0: 1}`, or you miss subarrays that start at index 0.
    - **ML lens:** counting time windows whose total crosses a threshold in event logs.
    - **In the wild:** `df.groupby(...).cumsum()` in pandas computes per-group prefix sums.
- **Capacity To Ship Packages.**
    - **Vary:** the answer lies between `max(weights)` and `sum(weights)`, and "can ship in D days" is monotonic in capacity. That monotonicity is what makes binary search legal.
    - **Break:** write a predicate that is not monotonic and watch the search return a wrong answer.
    - **ML lens and in the wild:** finding the largest batch size that fits in GPU memory. Hugging Face `accelerate`'s `find_executable_batch_size` in `src/accelerate/utils/memory.py` multiplies the batch size by 0.9 after each out-of-memory error. Why does it not binary search?
- **Insert Delete GetRandom O(1).**
    - **Break:** delete the last element, where the swap is with itself. Duplicates (LeetCode 381).
    - **ML lens:** a replay buffer that samples uniformly and deletes old items.
    - **In the wild:** since Python 3.11, `random.sample` no longer accepts a set. Why can't a hash set give a uniform random element in O(1)?
- **Path Sum III.**
    - **Break:** forget to decrement the prefix count when the DFS returns, and paths from other branches leak in. A deep, skewed tree hits the recursion limit.
    - **ML lens:** a decision tree's prediction is a root-to-leaf path. Path aggregates explain a prediction.
- **IPO.**
    - **Vary:** two heaps, or one sorted list plus one heap. Prove the greedy with an exchange argument, as in [Week 11](week_11.md).
    - **ML lens:** picking the next experiment to run under a compute budget, where finishing some experiments unlocks others.
- **Snakes and Ladders.**
    - **Break:** the board numbers rows in alternating directions, starting from the bottom. Most bugs are in the number-to-cell conversion. Test it alone, for every square on a 3 × 3 board.
    - **ML lens:** an agent searching a state space with BFS, where each move has the same cost.
- **Maximum Profit in Job Scheduling.**
    - **Vary:** this is the weighted interval scheduling from Week 11. Sort by end, and for each job find the last compatible job with `bisect` (`Lib/bisect.py`).
    - **Break:** a job that starts exactly when another ends. Is it compatible? Your `bisect_left` vs `bisect_right` choice decides.
- **Sum of Subarray Minimums.**
    - **Break:** with equal values, each subarray must be counted once. Use strict comparison on one side and non-strict on the other. Show the double count when both are strict.
    - **ML lens:** sliding-window minimum features in time series.
    - **Remember** the answer is modulo 10⁹ + 7. In Python nothing overflows, but you must still apply the modulo.

## ML/DL: capstone polish

### Learn

- **Error analysis by slice.** Group errors by user type, input length, language, or time. A good average can hide a slice that fails every time
- **Thresholds and calibration.** If your model outputs a score, pick the threshold on validation data for the cost of each error type. Check whether a score of 0.8 means right 80% of the time
- **One final test-set run.** Freeze everything, run the test set once, and report that number
- **A model card:** intended use, training data, evaluation, slices where it fails, and what it must not be used for
- **Serving trade-offs.** Merge the LoRA adapter for faster inference. Measure latency and memory before and after

### Read the code

**1. scikit-learn, `calibration_curve` in [`sklearn/calibration.py`](https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/calibration.py).**

- With `strategy="uniform"`, bins have equal width. With `"quantile"`, they have equal counts. When your model's scores bunch near 0 and 1, which strategy leaves empty bins?
- The docstring says bins with no samples are not returned. How could that mislead a plot?

**2. `huggingface_hub`, `ModelCard` in [`src/huggingface_hub/repocard.py`](https://github.com/huggingface/huggingface_hub/blob/main/src/huggingface_hub/repocard.py).** Which fields does the card's metadata hold, and which belong in the free-text body? Write your model card in this format even if you never upload it.

**3. `peft`, `merge_and_unload` in `src/peft/tuners/tuners_utils.py`.** After merging, can you still turn the adapter off? What does that mean for your fallback path?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `calibration_curve` | `n_bins` | `5` (a future release changes it; check the docs) | Number of bins | Many bins on a small test set: each bin is noise |
| `calibration_curve` | `strategy` | `"uniform"` | Equal-width vs equal-count bins | `"uniform"` with bunched scores leaves most bins empty |
| `classification_report` | `zero_division` | `"warn"` | Value used when a metric divides by zero | A class never predicted shows 0 precision with only a warning, easy to miss |
| `classification_report` | `digits` | `2` | Decimal places shown | Two digits hide a 0.004 change you might be claiming as a gain |
| `merge_and_unload` | `safe_merge` | `False` | Check merged weights for NaN | `False` can ship a model with NaN weights from a broken adapter |
| `merge_and_unload` | `adapter_names` | `None` | Which adapters to merge | `None` merges all active adapters, including one you forgot was active |

### Break it

Pick the decision threshold on the **test** set instead of validation, and report the score. Then pick it on validation and report the test score. The difference is how much the first number lied.

## AI agents: capstone polish

### Learn

- Triage from production traces (Weeks 12 and 13): group failures by type and count them
- Fix the top three failure types. Add a task to the eval set for each one before you fix it
- Freeze the eval set for launch. Compare every change against it
- Launch-day reliability: rate limits, a spend cap, a fallback when the provider is down, and a clear message when the agent cannot help
- A live demo with a recorded backup. If the live run fails, say so and switch to the recording. Never present a recording as live

### Experiment

On the frozen eval set, measure task success, cost per successful task, p95 latency, and handoff rate before and after your three fixes. Report each with an interval from three runs. Did any fix make another slice worse?

### Break it

- Send 10× your normal traffic at once. What fails first: your server, your rate limit, or your budget?
- Block access to your LLM provider. Does the fallback work, and does the user see a clear message?
- Give the product to someone who has never seen it, with no explanation. Where does the agent confuse them?

### ML lens

Agent triage is error analysis by slice, the same as the ML/DL track. A failure type that appears in only one user segment is a slice problem, not a prompt problem.

### In the wild

Your own production traces are this week's source. Compare their fields with the agent span attributes in `docs/gen-ai/gen-ai-agent-spans.md` in [open-telemetry/semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai), such as `gen_ai.agent.name` and `gen_ai.agent.id`. Which fields would have made this week's triage faster?

## Software dev: the launch checklist

### Learn

- A launch checklist turns "I think it's ready" into evidence. Each item gets a link, a screenshot, or a test
- Common items:
    - Domain and HTTPS
    - Sign-up and onboarding tested on a phone
    - Privacy policy, terms, and how a user deletes their data
    - Consent for any data you log or train on
    - Backups, and a **tested** restore
    - Rate limits and a spend cap
    - Error pages that say what to do next
    - Rollback tested this week (Week 12)
    - Secrets rotated, dependencies scanned, and the Week 10 security checks rerun
    - Monitoring and alerts on, and someone watching on launch day
    - A support channel users can find
    - README, docs, and license current (Week 13)

### Apply it

Write your checklist in the repo. A coding agent may draft it from this list. You add the items specific to your product, and you remove none without a written reason. Then work through every item.

### Verify

- Every item has evidence linked next to it.
- Restore from backup into a fresh environment and run your smoke test against it.
- A friend signs up on their own phone while you watch without helping. Fix what they trip on.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 92 | Mock: Subarray Sum Equals K | Error analysis by slice | Triage production traces | Write the launch checklist |
| 93 | Mock: Capacity To Ship Packages | Thresholds and calibration | Add eval tasks for the top 3 failures | Domain, HTTPS, onboarding on a phone |
| 94 | Mock: Insert Delete GetRandom | Read `calibration_curve`; fix the threshold | Fix failure 1; measure | Privacy, terms, data deletion, consent |
| 95 | Mock: Path Sum III | Model card | Fix failure 2; measure | Backups and a tested restore |
| 96 | Mock: IPO, Snakes and Ladders | Merge the adapter; latency and memory | Fix failure 3; freeze the eval set | Rate limits, spend cap, error pages |
| 97 | Mock: Max Profit in Job Scheduling | **Final test-set run** | 10× load; provider outage | Security rerun; secrets rotated |
| 98 | Mock: Sum of Subarray Minimums | Final numbers into the demo | Record the backup demo | Every checklist item has evidence |
| 99 | **Dress rehearsal:** full demo, timed, to a mentor and two participants. Fix what they flag | | | |
| 100 | **Demo day and public launch** | | | |

## Weekly build

### Demo day and launch

Your capstone, launched publicly, with a demo that shows real users and real evidence.

**Required behavior**

- **Product.**
    - Live at a public URL, with sign-up working for a stranger.
    - Every launch-checklist item done, with evidence.
- **Model (ML/DL).**
    - One final test-set number, slices where it fails, and a model card.
- **Agent (AI agents).**
    - The top three failure types fixed and measured on the frozen eval set.
    - A fallback for provider outages, and a spend cap.
- **DSA.**
    - 8 mock interviews done, with interviewer scores, and the depth ladder run on each solution. That brings you to 100 problems.
- **Demo.**
    - A 10-minute live demo (format below), a recorded backup, and a demo video of 3 minutes or less.

**Stretch**

- Launch on a community site where your users already gather, not only on LinkedIn.
- Offer your open-source component from Week 13 in the launch, with a "good first issue" ready for new contributors.

## Demo day (Day 100)

!!! note "Milestones by path"
    The milestone below is written for the Product and Academic papers paths. Your Day 100 milestone for each of your two paths is in [Choose your paths](../paths.md#milestones-by-path). The technical work in this week's build is the same for everyone.

| Milestone | Communication | Users and value |
|---|---|---|
| Day 100 | Demo day and public launch | Users, revenue, and lessons presented |

### The 10-minute format

| Minutes | Section | What to show |
|---|---|---|
| 1 | **Problem** | Who has it, in their words from your interviews |
| 2 | **Evidence of users** | Users, retention table, paying users or LOIs, one quote |
| 3 | **Live demo** | One real task, end to end. Switch to the recording if live fails, and say so |
| 2 | **Technical depth** | One hard decision: the alternatives, the measurement, and the result with an interval |
| 1 | **What AI did vs what you verified** | Agent PR acceptance rate, the eval gate, and one bad output you caught |
| 1 | **What's next** | Your next 30 days, with one number you will move |

Then 5 minutes of questions. Expect "why would they pay for this?" and "what breaks first at 10× users?".

### The launch

- Post the launch on Day 100, after the demo.
- Include the URL, what it does in one sentence, who it is for, the demo video, and one number (users, retention, or revenue).
- Ask for one specific action: try it, reply with a use case, or introduce one person who has the problem.
- **Patent route:** make sure the provisional was filed before the launch post, the demo video, and the live demo describe the invention. If in doubt, ask your patent agent before Day 100. See the [publishing and patents warning](../how-it-works.md).

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 92 | "A sliding window fails on negative numbers." The counterexample, and the prefix-sum fix. |
| 93 | "Why `accelerate` shrinks the batch by 10% instead of binary searching." |
| 94 | "Python 3.11 stopped letting `random.sample` take a set." Why a hash set has no O(1) random element. |
| 95 | "My model scored well overall and failed every time on one slice." The slice, and the fix. |
| 96 | "Merging my LoRA adapter cut p95 by X ms." Before and after, and what you lost. |
| 97 | "I picked the threshold on the test set and my score went up." How much the first number lied. |
| 98 | Your launch checklist, every item with evidence. |
| 99 | What the dress rehearsal exposed, and what you fixed. |
| 100 | **The launch post**, with the demo video. |

**This week's write-up: the launch post and a demo video.** The launch post goes out on Day 100. The demo video is 3 minutes or less: the problem, one real task end to end, and one number.

## Value-creation step

!!! note "Your paths"
    This step is written for the Product path. On other paths, use this time for your next milestone in [Choose your paths](../paths.md).

Launch publicly on Day 100, and present your users, revenue, and lessons on demo day. Then write your next 30 days in the repo:

- The one number you will move (users, retention, or revenue), and its value today.
- The three things you will do to move it.
- The users you will talk to, by name.
- What you will stop doing.

Day 100 is not the end. The next 30 days decide whether the capstone becomes a product.
