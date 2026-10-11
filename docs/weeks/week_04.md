# Week 4 (Days 22–28): Lists, Trees, Models, and the Agent Loop

## This week

- **You will learn:**
    - **DSA:** how linked lists and trees store structure in pointers, and when that beats an array.
    - **ML/DL:** how regression and classification models are trained, and which metrics tell the truth about them.
    - **AI agents:** how to write the agent loop yourself: call, run tools, repeat, and stop.
    - **Software dev:** how to put a model behind a REST API that validates its input.
- **Goal:** by Sunday you have a model trained on data from your own problem area, served over HTTP, and called by an agent you wrote without a framework.
- **Weekly build:** a model behind a REST API with a prediction cache, used as a tool by your own agent loop. See [Weekly build](#weekly-build).
- **AI rule (Days 21–60):** AI pairs, you prove. AI may write code. Write the tests first, and be ready to explain every line it wrote.

## DSA: linked lists and trees

**Focus.** A linked list gives up random access to get O(1) insert and delete once you hold the node. A tree is a linked structure that branches. Most tree problems are one recursion: solve the children, then combine at the node. Two things go wrong often: losing a pointer before you have saved it, and recursion that is deeper than Python allows.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 22 | [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | Three pointers: prev, curr, next |
| 22 | [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | Dummy head and two pointers |
| 23 | [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | Fast and slow pointers |
| 24 | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | Hash map plus doubly linked list |
| 25 | [104. Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) | Recurse, combine with max |
| 25 | [102. Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | Breadth-first with a queue |
| 26 | [543. Diameter of Binary Tree](https://leetcode.com/problems/diameter-of-binary-tree/) | Return height, record the best path |

### Depth ladder: LRU Cache

1. **Solve.** A dict from key to node, and a doubly linked list with sentinel head and tail nodes. `get` moves the node to the front. `put` inserts at the front and evicts from the back when over capacity. Both are O(1). Tests first: the LeetCode example, capacity 1, and `put` on an existing key that must update it, move it, and evict nothing.
2. **Vary.**
    - Rewrite it with `collections.OrderedDict`, `move_to_end`, and `popitem(last=False)`. How many lines? Benchmark both.
    - [460. LFU Cache](https://leetcode.com/problems/lfu-cache/). Evict the least frequently used key. What extra structure keeps it O(1)?
    - Entries expire after a time-to-live.
    - Capacity measured in bytes, not entries, for responses of different sizes.
3. **Break the assumptions.**
    - **Update at capacity.** If `put` evicts before checking whether the key already exists, it throws away a key it did not need to. Write the failing test.
    - **Capacity 0.** Does `put` crash, or store and immediately evict?
    - **A scan.** Read 2 × capacity distinct keys once each, then go back to your hot keys. The hit rate collapses. This is the classic weakness of LRU.
    - **Threads.** Two threads move the same node at once. What state can the list end up in?
    - **Memory.** Measure 10⁶ entries with `tracemalloc` for your node class, a node class with `__slots__`, and `OrderedDict`.
4. **ML lens.**
    - Cache predictions keyed by a hash of the input. The key must include the **model version**, or you serve stale predictions after a redeploy. You build this cache this week.
    - Agents repeat tool calls. Caching results for identical calls saves tokens and latency.
    - LLM inference servers keep a KV cache (Week 10) and evict cached prefixes when memory runs out.
5. **In the wild.**
    - `functools.lru_cache` in CPython `Lib/functools.py`. Read `_lru_cache_wrapper`. The pure-Python version stores each link as a 4-item list, `[PREV, NEXT, KEY, RESULT]`, around one `root` sentinel. Why lists, not node objects? What does `typed=True` change? Find the line that swaps in the C version from `_functools`.
    - `OrderedDict` in `Lib/collections/__init__.py` keeps a doubly linked list of `_Link` objects next to the dict. Read the comment about `self.__map`.

### Depth ladder: Linked List Cycle

1. **Solve.** Floyd's tortoise and hare: slow moves one step, fast moves two. They meet if and only if there is a cycle. O(n) time, O(1) space. Compare with a set of visited nodes: O(n) space. Tests first: empty list, one node, a self-loop, a cycle back to the head, no cycle.
2. **Vary.**
    - [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/). Return where the cycle starts. Prove why resetting one pointer to the head works.
    - Return the cycle's length.
    - [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/). Treat the array as a linked list where `i` points to `nums[i]`.
3. **Break the assumptions.**
    - **Values vs identity.** A set of node *values* reports a cycle in `1 → 2 → 1 → None`. Use node identity.
    - **`id()` reuse.** CPython can reuse an object's `id()` after the object is freed. When could a visited set of ids give a false positive?
    - **Graphs.** A node with two outgoing edges breaks Floyd. General cycle detection needs DFS with node colors (Week 7).
4. **ML lens.**
    - Agents loop. The model calls the same tool with the same arguments, gets the same result, and calls it again. Hash `(tool, arguments)` and stop on a repeat. You add this to your agent loop this week.
    - Autograd's topological sort (Week 3) assumes the graph has no cycles. RNNs work by unrolling the cycle over time steps.
5. **In the wild.**
    - `copy.deepcopy` in CPython `Lib/copy.py` keeps a `memo` dict keyed by `id()`. Make a list that contains itself and deep-copy it. Why does it not recurse forever?
    - `json.dumps` in `Lib/json/encoder.py` keeps a `markers` dict and raises `ValueError: Circular reference detected`. What happens with `check_circular=False`?

### Depth ladder: Reverse Linked List, Merge Two Sorted Lists

- **Vary:** reverse recursively; reverse in groups of k. Merge [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) with a heap.
- **Break:** the recursive reverse on 10⁵ nodes raises `RecursionError`. When two nodes have equal keys, does your merge keep them in their original order? Write the test.
- **ML lens:** merging sorted runs is how you sort a dataset larger than memory: sort chunks, write them out, merge.
- **In the wild:** the Linux kernel's `include/linux/list.h` puts a `list_head` inside each data struct, rather than the data inside a list node. Why does that let one object sit on several lists at once?

### Depth ladder: Maximum Depth, Level Order

- **Vary:** write both iteratively: depth-first with an explicit stack, breadth-first with `collections.deque`.
- **Break:** a degenerate tree, where every node has only a right child, is a linked list. At depth 10⁵ the recursive version fails. A list used as a queue with `pop(0)` makes BFS O(n²). Measure it.
- **ML lens:** a decision tree's `max_depth` (Week 5) limits how finely it can split the data. Deeper trees fit training data better and overfit faster.
- **In the wild:** `ast.walk` in CPython `Lib/ast.py` uses a `deque`, so it is breadth-first. `ast.NodeVisitor.generic_visit` recurses. Print the order each one visits the nodes of `a + b * c`.

### Day 26 extras: Diameter of Binary Tree

- **Solve:** a post-order traversal that returns each subtree's height and updates the best `left + right` seen so far. O(n) time.
- **Break:**
    - Is the diameter counted in edges or in nodes? Off-by-one here passes most examples.
    - A degenerate tree of 10⁴ nodes in a chain exceeds Python's default recursion limit of 1000. Rewrite it with an explicit stack.
- **ML lens:** tree height bounds prediction cost. A decision tree with `max_depth=None` can grow as deep as the data allows. Fit one in scikit-learn and compare `get_depth()` with `get_n_leaves()`.
- **In the wild:** SQLite `src/btree.c`. A B-tree keeps many keys per node, so it stays shallow. Why is that better than a binary tree when each node is a disk page?

## ML/DL: regression, classification, and metrics

### Learn

- Linear regression: mean squared error, the closed form with `lstsq` (Week 2), and gradient descent (Week 3)
- Logistic regression: the sigmoid, binary cross-entropy, and the decision threshold ([cross-entropy](https://dpaul0501.github.io/llm-agent-reads/part_05_training_and_optimization/))
- L2 regularization. In scikit-learn, `C` is the *inverse* of its strength.
- Train, validation, and test splits. Stratification. A baseline from `DummyClassifier` before any real model.
- Confusion matrix, precision, recall, F1, ROC AUC, average precision (PR AUC), and calibration
- Choosing a threshold from the cost of a false positive vs a false negative

### Read the code

**1. scikit-learn `sklearn/metrics/_classification.py`.** Find `precision_recall_fscore_support`. Answer in your log:

- What does `precision_score` return when the model predicts no positives at all? Find the `zero_division` handling.
- Compute `average="micro"`, `"macro"`, and `"weighted"` on an imbalanced 3-class example. Which one hides the rare class? Show that `"micro"` precision equals accuracy for single-label multiclass.

**2. scikit-learn `sklearn/metrics/_ranking.py`.** `roc_curve` and `precision_recall_curve` call a helper that sorts scores with a stable descending `argsort` and takes cumulative sums (`confusion_matrix_at_thresholds` in 1.9; `_binary_clf_curve` in older versions). This is Week 2's sort-and-sweep.

- Apply `np.log` to positive scores, or any other strictly increasing function. Does ROC AUC change? Why not?
- `roc_curve` defaults to `drop_intermediate=True`. Which points does it drop, and does that change the AUC?

**3. scikit-learn `sklearn/linear_model/_logistic.py`.** `LogisticRegression`.

- Fit on unscaled features with very different ranges. Do you get a `ConvergenceWarning`? Scale with `StandardScaler` and fit again.
- `predict` returns class 1 when `decision_function` is above 0. What probability does that match in `predict_proba`?
- Recent versions deprecate `penalty` in favor of `l1_ratio`. Check what your installed version does.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `LogisticRegression` | `C` | `1.0` | Inverse of regularization strength | Tiny: coefficients near 0, every prediction near the base rate. Huge on separable data: weights grow and probabilities become overconfident |
| `LogisticRegression` | `max_iter` | `100` | Solver iterations | On unscaled features, a `ConvergenceWarning` and coefficients that are not at the optimum |
| `LogisticRegression` | `class_weight` | `None` | Per-class weight in the loss | `"balanced"` raises minority recall but shifts probabilities, so they are no longer calibrated |
| `train_test_split` | `test_size` | `None` (0.25 when `train_size` is also `None`) | Share of data held out | Small test set: the metric's error bar is wide (Week 2) |
| `train_test_split` | `stratify` | `None` | Keep class ratios in both splits | A rare class can be missing from the test set entirely |
| `train_test_split` | `shuffle`, `random_state` | `True`, `None` | Random order, and its seed | Time-ordered data: shuffling puts the future in training. No seed: numbers change every run |
| `precision_score` | `average` | `"binary"` | How per-class scores combine | `"macro"` weighs a 10-example class as much as a 10,000-example class |
| `precision_score` | `zero_division` | `"warn"` | Value when there are no predicted positives | A model that never predicts positive gets 0.0 and a warning that is easy to ignore |
| `roc_curve` | `drop_intermediate` | `True` | Drop points that do not change the curve's shape | Fewer thresholds to choose from when you pick an operating point |
| `DummyClassifier` | `strategy` | `"prior"` | What the baseline predicts | On 99:1 data it scores 99% accuracy with zero recall |

### Break it

Make a 99:1 dataset with `make_classification(weights=[0.99], random_state=0)`. Report accuracy, recall, ROC AUC, and average precision for `DummyClassifier` and for `LogisticRegression`. Then compare the threshold of 0.5 with a threshold chosen on the validation set for your stated costs. Which metric would have fooled you, and by how much?

## AI agents: the agent loop from scratch

### Learn

- **The loop.** Send messages and tool definitions. If the response has tool calls, validate them, run them, append the results, and call again. Otherwise return the answer.
- **State is the message list.** Every step sends the whole history again.
- **Stop conditions:** a final answer, max steps, a token budget, a wall-clock timeout, and a repeated identical call
- **Error policy:** retry transient API errors with backoff (Week 1). Return tool errors to the model (Week 3). Stop on anything else.
- **Traces.** Log each step: the request, tool calls, tool results, tokens, and latency, as one JSON line per step.

### Experiment

Write 20 tasks that need between one and four tool calls. Run your loop with `max_steps` set to 2, 4, and 8. Measure success rate, steps used, and total input tokens per task. Plot cumulative input tokens against step number for one long task. Because each step resends the history, total input tokens grow roughly with the square of the number of steps. Does your plot show it?

### Break it

- **A tool that always fails.** Does the loop stop with a clear reason, or burn all its steps?
- **Repeats.** Script the model to call the same tool with the same arguments three times. Does your repeat detector stop it?
- **A huge tool result** pushes the history past the context limit. What error comes back, and what does your loop do with it?
- **Truncated arguments.** The stop reason is max tokens in the middle of a tool call. The arguments are half a JSON object. Do you run the tool anyway?

### ML lens

If each step succeeds with probability p, independently, an n-step task succeeds with probability pⁿ. With p = 0.95, ten steps succeed only about 60% of the time. Compare that with your measured success rate by number of steps. Then treat "did the model call the right tool" as a classifier and compute its precision and recall with this week's metrics.

### In the wild

Hugging Face `smolagents`, `src/smolagents/agents.py`. Find the `while` loop in `_run_stream` and the default `max_steps: int = 20`. Read `_handle_max_steps_reached`. When steps run out, it asks the model for a final answer anyway. Is that what you want for your users, or should the loop report failure?

## Software dev: REST APIs

### Learn

- Resources and methods: `GET` reads, `POST` acts or creates. Which methods are idempotent, and why it matters for retries.
- Status codes: 200, 201, 400, 404, 422, 429, 500, 503
- Request and response schemas with Pydantic. FastAPI generates an OpenAPI schema at `/openapi.json` and docs at `/docs`.
- Load the model once at startup with a lifespan handler, not on every request
- Versioned paths (`/v1/predict`) and a health endpoint
- Testing with `fastapi.testclient.TestClient`

### Apply it

Serve this week's model with three endpoints: `GET /health`, `GET /v1/model-info` (version, metrics, threshold), and `POST /v1/predict` (one or many rows). Write the TestClient tests before the handlers.

### Verify

- Tests show 200 for valid input, and 422 for a missing feature and for a non-numeric string where a number belongs. (Pydantic converts `"3"` to `3` by default.)
- Benchmark 1,000 requests and report p50 and p95 latency. Then move model loading into the request handler and benchmark again. That is the seeded bug, and your benchmark should catch it.
- Commit `openapi.json`. Rename a field and check that the diff shows up in review.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 22 | Reverse List, Merge Two Lists | Linear regression: `lstsq` vs gradient descent | The loop: call, tool, result, repeat | HTTP methods, status codes |
| 23 | Linked List Cycle | Logistic regression, BCE; read `_logistic.py` | Stop conditions: steps, budget, repeats | FastAPI and Pydantic schemas |
| 24 | LRU Cache | Confusion matrix, precision, recall; read `_classification.py` | Step traces as JSON lines | Lifespan loading, TestClient |
| 25 | Max Depth, Level Order | ROC and PR curves; read `_ranking.py` | `max_steps` experiment and token growth | Error responses: 422, 503 |
| 26 | Diameter of Binary Tree | Imbalance, thresholds, Break it | Break it: failing tools, repeats, truncation | Latency benchmark, p50 and p95 |
| 27 | **Weekly build** | | | |
| 28 | **Defend and publish** | | | |

## Weekly build

### A model behind an API, used as an agent tool

```bash
python -m model.train data/cases.csv --target label --out artifacts/
uvicorn app.main:app --port 8000
curl -X POST localhost:8000/v1/predict -H 'Content-Type: application/json' -d @example.json
python -m agent "Which of these five cases is most at risk, and why?"
```

**Use your own problem area.** Find or collect a small tabular dataset that fits the problem from your Week 2 statement. Use a public dataset only if nothing closer exists, and say in the README how it relates to your problem.

**Required behavior**

- **Model (ML/DL).**
    - A `DummyClassifier` baseline and a logistic regression (or linear regression for a numeric target), on a stratified split.
    - Precision, recall, F1, ROC AUC, average precision, and the confusion matrix on a held-out test set.
    - A threshold chosen on the validation set from costs you state.
    - The model, its version, its threshold, and its metrics saved together.
- **API (Software dev).**
    - FastAPI with `/health`, `/v1/model-info`, and `/v1/predict` for one or many rows.
    - Pydantic validation, 422 on bad input, and the model loaded once at startup.
- **Cache (DSA).**
    - An LRU cache for predictions, written from scratch with a dict and a doubly linked list.
    - Keyed by model version and the canonicalized input.
    - `/v1/cache-stats` reports hits, misses, and size.
- **Agent (AI agents).**
    - Your own loop, with no agent framework.
    - Tools: `predict`, which calls your API over HTTP, and `model_info`.
    - `max_steps`, a token budget, and repeat detection. One trace file per run.
- **Engineering (Software dev).**
    - TestClient tests for the API.
    - Agent tests with a fake model client that scripts tool calls.
    - A README with setup, an example run, and the metrics table.

**Stretch**

- A calibration plot with `CalibrationDisplay`. Is your model's 0.8 really 80%?
- Compare with a decision tree, ahead of [Week 5](week_05.md).

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 22 | "The Linux kernel puts the linked list inside the data." Walk through `list_head` in `include/linux/list.h`. |
| 23 | "How `json.dumps` knows your object contains itself." Show the `markers` dict in `encoder.py`. |
| 24 | "`functools.lru_cache` builds its linked list out of 4-item Python lists." Show `PREV, NEXT, KEY, RESULT`. |
| 25 | "My model had a good ROC AUC and a poor average precision. Both are true." Show the 99:1 example. |
| 26 | "0.95¹⁰ ≈ 0.60: my agent's success rate by number of steps." Measured vs predicted. |
| 27 | Your agent calling your model's API, with one trace shown step by step. |
| 28 | What the defense exposed, plus one anonymized pattern from your interviews. |

**This week's write-up:** a Kaggle notebook on a public dataset close to your problem: the baseline, the logistic regression, the PR curve, and how you chose the threshold from costs. A public score counts if the dataset has a competition.

## Value-creation step

!!! note "Your paths"
    This step is written for the Product path. On other paths, use this time for your next milestone in [Choose your paths](../paths.md).

Run at least three of your five interviews this week, using the questions from [Week 3](week_03.md). Write notes within 24 hours. Record what people did, what it cost them, and what they have tried, not what they say they would do. Finish the remaining interviews by Day 30, when the Month 1 milestone asks for a summary of all five.
