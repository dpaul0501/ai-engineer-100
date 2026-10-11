# Week 12 (Days 78–84): Weighted Graphs, Honest Evaluation, and Production

## This week

- **You will learn:**
    - **DSA:** shortest paths with Dijkstra, connectivity with union-find, ordering with topological sort, and minimum spanning trees.
    - **ML/DL:** how to design an experiment whose result you can trust, and how to put an error bar on it.
    - **AI agents:** where an agent's cost and latency go, and how to cut them without losing quality.
    - **Software dev:** how to deploy with a rollback, and how to monitor what users actually experience.
- **Goal:** by Sunday your capstone runs in production for real users, you can show its cost and p95 latency per task, and every claim of "better" comes with a confidence interval.
- **Weekly build:** the capstone, deployed, monitored, and measured. See [Weekly build](#weekly-build).
- **AI rule (Days 61–100):** AI agents, you review. Agents write code. You write the spec, review the PR, run the evals, and reject bad output.

## DSA: advanced graphs

**Read first:** the Graphs rows of [DSA to ML](https://mldsa.github.io/docs/index-dsa-to-ml/) in *From LeetCode to Learning Systems*.

**Focus.** Week 7's BFS treats every edge as cost 1. Real graphs have weights, and the questions get sharper. Shortest path: Dijkstra, when weights are non-negative. Are these two in the same group: union-find. In what order can I do these steps: topological sort. What is the cheapest way to connect everything: minimum spanning tree. Each one is a short algorithm with one assumption that, when broken, gives a wrong answer without an error.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 78 | [743. Network Delay Time](https://leetcode.com/problems/network-delay-time/) | Dijkstra with a heap |
| 79 | [787. Cheapest Flights Within K Stops](https://leetcode.com/problems/cheapest-flights-within-k-stops/) | Bellman–Ford, k + 1 rounds |
| 79 | [1631. Path With Minimum Effort](https://leetcode.com/problems/path-with-minimum-effort/) | Dijkstra with `max` instead of `+` |
| 80 | [684. Redundant Connection](https://leetcode.com/problems/redundant-connection/) | Union-find: the first edge inside one set |
| 80 | [547. Number of Provinces](https://leetcode.com/problems/number-of-provinces/) | Count the roots |
| 81 | [210. Course Schedule II](https://leetcode.com/problems/course-schedule-ii/) | Kahn's algorithm, in-degree queue |
| 82 | [1584. Min Cost to Connect All Points](https://leetcode.com/problems/min-cost-to-connect-all-points/) | Prim on a dense graph |

### Depth ladder: Network Delay Time

1. **Solve.** A min-heap of `(distance, node)`. Pop the closest node. If you have already finalized it, skip it. Otherwise finalize it and push its neighbors. The answer is the largest final distance, or `-1` if some node is unreachable. O((V + E) log V). Write tests for the given examples first.
2. **Vary.**
    - One target only. Stop when you pop it.
    - All pairs with `n ≤ 100`. Floyd–Warshall in O(n³). When is that simpler than running Dijkstra `n` times?
    - Negative edges. Bellman–Ford in O(VE), and it detects negative cycles.
    - Return the path, not only the distance. Store a predecessor per node.
    - A* with a heuristic. What property must the heuristic have so the answer stays correct?
3. **Break the assumptions.**
    - **A negative edge.** Build a 3-node graph where Dijkstra finalizes a node too early and returns the wrong distance. Does your code notice? Now run the same graph through networkx's Dijkstra and read the error, if any.
    - **Ties in the heap.** Push `(dist, node)` where the node is a dict or a custom object. Two equal distances force Python to compare the nodes, and you get a `TypeError`. How does networkx avoid this?
    - **Stale entries.** Python's `heapq` has no decrease-key, so you push duplicates and skip them on pop. On a dense graph, how large can the heap get?
    - Float weights. Two paths with equal true cost differ in the last bit. Is your output deterministic?
4. **ML lens.**
    - **Search in decoding and planning.** Beam search and A* are both best-first search over partial sequences. An agent planner choosing tool calls by cost (tokens, latency) is shortest path over states. This week's agents track measures those costs.
    - **Viterbi** is a shortest path in a DAG. It decodes the best tag sequence in a CRF and the best segmentation in a unigram tokenizer.
    - **Knowledge-graph retrieval.** Multi-hop questions become a shortest path between two entities.
5. **In the wild.**
    - networkx: `_dijkstra_multisource` in `networkx/algorithms/shortest_paths/weighted.py`. Find the `count()` used as a tie-breaker (`heappush(fringe, (0, next(c), source))`), the `if v in dist: continue` that skips stale entries, the `cutoff` argument, and the `ValueError` that says "Contradictory paths found: negative weights?". When does that check fire, and when can a negative edge slip through?
    - SciPy: `scipy.sparse.csgraph.dijkstra`. Benchmark it against yours and against networkx at 10⁵ edges.

### Depth ladder: Redundant Connection

1. **Solve.** Process edges in order. If both ends already have the same root, that edge closes a cycle: return it. Otherwise union the two sets. With path compression and union by size, each operation is nearly O(1) (inverse Ackermann).
2. **Vary.**
    - The graph is directed (LeetCode 685, Redundant Connection II). Why does plain union-find no longer suffice?
    - Count the components after each edge. Each successful union lowers the count by one. That is Number of Provinces as a stream.
    - Edges are deleted, not added. Union-find cannot split a set. Process the operations offline in reverse.
3. **Break the assumptions.**
    - Remove path compression and union by size, then feed a chain `1-2, 2-3, ..., n-1-n` with `n = 10⁵`. Time it. Each find is now O(n).
    - A recursive `find` on that chain hits Python's recursion limit. Write it iteratively.
    - Nodes are strings, not `1..n`. Use a dict for the parent map, and create entries lazily.
4. **ML lens.**
    - **Deduplication clusters.** Week 1's MinHash LSH gives you pairs of near-duplicate documents. Union-find turns pairs into clusters. Keep one document per cluster.
    - **Entity resolution.** "These two records are the same customer" is a union. Clusters are entities.
    - **Clustering.** Single-linkage clustering is Kruskal's algorithm stopped early. scikit-learn's HDBSCAN builds a minimum spanning tree: see `mst_from_mutual_reachability` and `make_single_linkage` in `sklearn/cluster/_hdbscan/_linkage.pyx`.
5. **In the wild.**
    - networkx: `networkx/utils/union_find.py`. `__getitem__` is the find. It walks to the root, then compresses the path in a second loop, with no recursion. `union` attaches lighter trees to the heaviest root. New elements are created on first lookup. Compare each choice with yours.
    - The same class is used by `kruskal_mst_edges` in `networkx/algorithms/tree/mst.py`.

### Depth ladder: Course Schedule II

1. **Solve.** Kahn's algorithm. Count in-degrees, queue every node with in-degree 0, pop, append to the order, and decrement its neighbors. If the order has fewer than `n` nodes, there is a cycle: return `[]`. O(V + E).
2. **Vary.**
    - The lexicographically smallest valid order. Replace the queue with a heap.
    - The minimum number of semesters if you can take any number of courses at once. Process the queue level by level.
    - DFS-based topological sort with three colors (unvisited, in progress, done). Where does it detect a cycle?
3. **Break the assumptions.**
    - Duplicate prerequisite pairs. If your adjacency is a set but your in-degree counts duplicates, some nodes never reach 0. Write the test.
    - A self-loop `[a, a]`. A course with no edges at all must still appear in the output.
    - A DFS version on a 10⁴-node chain hits the recursion limit.
4. **ML lens.**
    - **Autograd.** The backward pass runs in reverse topological order of the computation graph. That is what your Week 3 engine did.
    - **Pipelines.** Training and data pipelines are DAGs. A scheduler runs every step whose inputs are ready.
    - **Parallel tool calls.** An agent plan where some tool calls depend on others is a DAG. Run every call with in-degree 0 at the same time. You use this in this week's latency work.
5. **In the wild.**
    - CPython: `TopologicalSorter` in `Lib/graphlib.py`. Read `prepare`, `get_ready`, `done`, and `static_order`, and the `CycleError` it raises. Why is the API split into `get_ready()` and `done()` instead of returning one list? Use it to run your agent's independent tool calls in parallel.

### Depth ladder: Cheapest Flights Within K Stops

- **Vary:** run Bellman–Ford for `k + 1` rounds. Or run Dijkstra on the state `(city, stops used)`.
- **Break:** update the distance array in place during a round, and a path can use more than `k + 1` edges in one round. Copy the array each round. Plain Dijkstra on cities alone is also wrong here: show a case where the cheapest path to a middle city uses too many stops.
- **ML lens:** a constrained shortest path is planning with a budget. "Cheapest answer within 3 tool calls" has the same shape.

### Depth ladder: Path With Minimum Effort

- **Vary:** the cost of a path is its largest step, not the sum. Dijkstra still works because `max` never decreases along a path. Two other solutions: binary search on the answer plus BFS, and sorting edges plus union-find until start and end connect. Implement two of the three and compare.
- **Break:** a grid of 1 × 1. All heights equal.
- **ML lens:** minimax paths are bottleneck paths. The weakest link decides, as in a pipeline whose throughput is its slowest stage.

### Day 80 extra: Number of Provinces

- **Vary:** solve it with union-find and with DFS. On an adjacency matrix, both read all n² cells. Which is simpler to get right?
- **ML lens:** threshold a similarity matrix of embeddings and count the components. That is a crude clustering. How sensitive is the count to the threshold?

### Day 82 extra: Min Cost to Connect All Points

- **Vary:** the graph is complete, so E = n². Prim with an array is O(n²). Kruskal sorts n² edges, O(n² log n). Benchmark both at n = 1,000.
- **Break:** two identical points. Manhattan distance can be 0.
- **ML lens:** an MST is the skeleton of single-linkage clustering. Cut its k − 1 longest edges and you have k clusters.
- **In the wild:** `kruskal_mst_edges`, `prim_mst_edges`, and `boruvka_mst_edges` in `networkx/algorithms/tree/mst.py`. Why would a library offer three?

## ML/DL: experiment design and evaluation

### Learn

- Write the hypothesis and the decision before you run: "If B beats A by more than X on metric M, I ship B"
- One change at a time, against a fixed baseline
- Splits: train, validation, test. Touch the test set once
- k-fold, stratified k-fold, group k-fold, and time-series splits. Pick by how the data will arrive in production
- Leakage: duplicates across splits, the same user in two splits, features computed from the future
- **Variance.** Rerun with several seeds. A gain smaller than the seed-to-seed spread is not a gain
- **Confidence intervals** with the bootstrap. **Paired** comparisons when both systems run on the same examples
- Effect size vs significance. Many comparisons inflate false positives
- Offline eval vs online behavior. Why a model that wins offline can lose with users

### Read the code

**1. scikit-learn, [`sklearn/model_selection/_split.py`](https://github.com/scikit-learn/scikit-learn/blob/main/sklearn/model_selection/_split.py).** Answer in your log:

- `train_test_split` is a wrapper around `ShuffleSplit`. With `test_size=None` and `train_size=None`, what fraction goes to test? Find it in the docstring.
- `KFold` defaults to `shuffle=False`. On data sorted by label, what does the first fold look like? Find the `ValueError` raised when you pass `random_state` with `shuffle=False`. Why does scikit-learn refuse rather than ignore it?
- `StratifiedKFold._make_test_folds`: how does it keep class proportions in each fold?
- `TimeSeriesSplit` has a `gap` parameter. What leakage does a gap prevent?

**2. SciPy, `bootstrap` in `scipy/stats/_resampling.py`**, and **EleutherAI lm-evaluation-harness, `bootstrap_stderr` in [`lm_eval/api/metrics.py`](https://github.com/EleutherAI/lm-evaluation-harness/blob/main/lm_eval/api/metrics.py).**

- lm-eval resamples in chunks of `min(1000, iters)` and runs `iters // chunk_size` chunks. Pass `iters=1500`. How many resamples does it really draw?
- Read the comment in `bootstrap_stderr` about bias. What does it say the estimate is missing, and why do the authors accept it?
- `stderr_for_metric` uses a closed form for `mean` and the bootstrap for `f1_score` and others. Why can't F1 use the closed form?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `train_test_split` | `test_size` | `None` (0.25 if `train_size` is also `None`) | Size of the held-out set | Too small: a wide interval on the test score. Too large: less to train on |
| `train_test_split` | `shuffle` | `True` | Shuffle before splitting | `True` on time-ordered data leaks the future into training |
| `train_test_split` | `stratify` | `None` | Keep class proportions | `None` on a rare class can leave the test set with almost none of it |
| `KFold`, `StratifiedKFold` | `n_splits` | `5` | Number of folds | 2: high-variance estimate. `n` (leave-one-out): slow, and still high variance |
| `KFold`, `StratifiedKFold` | `shuffle` | `False` | Shuffle before folding | `False` on sorted data gives folds that differ systematically |
| `TimeSeriesSplit` | `gap` | `0` | Samples dropped between train and test | `0` with lagged features leaks the boundary |
| `scipy.stats.bootstrap` | `n_resamples` | `9999` | Number of resamples | Too few: the interval itself is noisy |
| `scipy.stats.bootstrap` | `confidence_level` | `0.95` | Width of the interval | 0.99 on 50 examples gives an interval too wide to decide anything |
| `scipy.stats.bootstrap` | `method` | `'BCa'` | How the interval is computed | `'percentile'` is simpler but can be biased on skewed statistics |
| `scipy.stats.bootstrap` | `paired` | `False` | Resample pairs together | `False` when comparing two systems on the same examples throws away the pairing and widens the interval |

### Break it

Take one model and one eval set. Run it with five seeds. Report the spread. Now pick the best seed and the worst seed and present them as "baseline" and "improved". Compute the gap. That gap is how large a fake improvement can look. Every comparison you make this week must clear it.

## AI agents: production agents, cost and latency

### Learn

- Where the tokens go: system prompt, tool schemas, conversation history, retrieved context, tool results
- **Cost per successful task**, not cost per call. Retries and failed runs count
- Latency: time to first token, total time, and time per step. p50 and p95
- **Prefix caching.** Many providers reuse computation for a repeated prompt prefix. Put stable content first and changing content last. Check your provider's docs for how it works and what it costs
- **Routing.** Send easy requests to a cheaper path, such as your Week 11 fine-tuned model, and escalate the rest
- Streaming the answer so users see progress
- Parallel tool calls for independent steps
- Budgets: a step limit, a token limit, and a timeout per request. What the agent does when it hits one

### Experiment

Replay 100 real tasks from your capstone logs (with consent, and with personal data removed). For each task, log tokens by component, latency per step, total cost, and whether it succeeded. Then make three changes, **one at a time**, and measure each against the same 100 tasks:

- Reorder the prompt so the stable part is a fixed prefix.
- Trim tool schemas and history to what the task needs.
- Route easy tasks to your fine-tuned model.

For each change, report the change in cost per successful task and in p95 latency, with a paired bootstrap interval from the ML/DL track. Keep a change only if quality does not drop beyond your stated threshold.

### Break it

- Give the agent a task it cannot complete, with no step limit. How many tokens does it burn before you stop it?
- Make one tool time out. Does the agent retry with the full context each time? Count the cost of the retry storm.
- Put the current timestamp at the top of the system prompt. What happens to your prefix-cache hit rate, if your provider reports one?

### ML lens

Prefix caching works because of the KV cache from Week 10: the keys and values for a fixed prefix are the same every time, so they can be stored and reused. Attention cost grows with context length, so trimming context saves both money and time. See the KV cache section of the [inference chapter](https://dpaul0501.github.io/llm-agent-reads/part_06_inference_and_tricks/).

### In the wild

- LiteLLM: `litellm/router.py` in [BerriAI/litellm](https://github.com/BerriAI/litellm). Find how it handles fallbacks, retries, and cooldowns for a failing deployment. When does it stop retrying?
- vLLM: `schedule()` in `vllm/v1/core/sched/scheduler.py` spends a `token_budget` per step. Your agent needs a budget for the same reason.
- OpenTelemetry GenAI semantic conventions: [open-telemetry/semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai), `docs/gen-ai/gen-ai-metrics.md`. Which metrics does it define for duration and token usage, and what histogram bucket boundaries does it recommend?

## Software dev: deploy and monitor

### Learn

- Config through environment variables. Secrets never in the repo
- Health and readiness endpoints
- One-command deploys from CI on merge to `main`. A tested rollback
- Database migrations that can run forward and back
- Metrics: request rate, error rate, and duration (the RED method). Plus tokens and cost per task for an AI product
- Alert on what users feel (errors, slow responses) and on spend, not on CPU alone
- Read the source: the `Histogram` class in `prometheus_client/metrics.py` in [prometheus/client_python](https://github.com/prometheus/client_python). Its `DEFAULT_BUCKETS` stop at 10 seconds before `+Inf`. Where does a 15-second agent run land? What does that do to your p95? Compare with the GenAI bucket boundaries above.

### Apply it

Deploy your capstone to a host of your choice.

- CI deploys on merge to `main`. A coding agent may write the pipeline from your spec. You review it.
- Health check, and rollback to the previous version with one command.
- Metrics: request rate, error rate, a latency histogram with buckets that fit your real latencies, tokens and cost per task, and success rate.
- Two alerts: error rate above your threshold, and daily spend above your cap.

### Verify

- Deploy a build with a seeded bug that fails the health check. Show that the deploy stops or rolls back.
- Fire a synthetic alert and show that it reaches you.
- Send 50 requests with known latencies and check that your dashboard's p95 matches the true p95.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 78 | Network Delay Time | Hypothesis, baseline, one change at a time | Log tokens by component for 100 tasks | Config, secrets, health endpoint |
| 79 | Cheapest Flights, Minimum Effort | Splits; read `_split.py` | Cost per successful task; p50 and p95 | CI deploy on merge; rollback |
| 80 | Redundant Connection, Provinces | Seeds and variance; the fake-improvement gap | Prefix ordering; measure the change | Metrics; read `prometheus_client` Histogram |
| 81 | Course Schedule II | Bootstrap; read lm-eval `bootstrap_stderr` | Parallel tool calls with `graphlib` | Alerts on errors and spend |
| 82 | Min Cost to Connect Points | Paired comparison of this week vs last week | Routing to the fine-tuned model | Seeded bad deploy; dashboard check |
| 83 | **Weekly build** | | | |
| 84 | **Defend and publish** | | | |

## Weekly build

### Capstone in production

Your capstone, live, with real users, and with numbers you can defend.

**Required behavior**

- **Deploy (Software dev).**
    - Deployed from CI on merge to `main`, with a health check and a one-command rollback.
    - Secrets in environment variables only.
- **Monitor (Software dev + AI agents).**
    - A dashboard with request rate, error rate, p50 and p95 latency, success rate, and cost per successful task.
    - Alerts on error rate and daily spend.
- **Cost and latency (AI agents).**
    - Step, token, and time budgets on every agent run, and a defined behavior when a budget runs out.
    - At least one measured optimization (prefix ordering, trimming, or routing), kept only if quality held.
- **Graphs where they fit (DSA).**
    - Use one of this week's algorithms in the product. Examples: run independent tool calls in topological order with `graphlib`, or cluster duplicate user feedback with union-find.
- **Evaluation (ML/DL).**
    - An offline eval comparing this week's version with last week's, on the same examples, with a paired bootstrap interval.
    - A short experiment log: hypothesis, change, result, decision.
- **Engineering.**
    - A README section on how to deploy, roll back, and read the dashboard.

**Stretch**

- A canary deploy: send 10% of traffic to the new version and compare error rates before full rollout.
- A load test at 10× your current traffic. Which part fails first?
- A per-user cost report, so you know what each paying user would cost to serve.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 78 | "Dijkstra with one negative edge." The 3-node graph, the wrong answer, and what networkx does about it. |
| 79 | "Why networkx pushes a counter into its heap." Tie-breaking and the `TypeError` it avoids. |
| 80 | "My 'improvement' was seed noise." The best-seed vs worst-seed gap. |
| 81 | "`graphlib` ran my agent's tool calls in parallel." Latency before and after. |
| 82 | "Prometheus's default buckets stop at 10 seconds." What that did to my agent's p95. |
| 83 | The capstone in production: cost per successful task and p95, with intervals. |
| 84 | What the defense exposed, and what you fixed. |

**This week's write-up: GitHub.** A README for your production capstone that a stranger can follow: architecture, how to deploy and roll back, what the dashboard shows, and eval results with confidence intervals.

If you plan to file a provisional patent in [Week 13](week_13.md), keep the core idea out of the public README and posts. Describe the deployment, monitoring, and evaluation instead. See the [publishing and patents warning](../how-it-works.md).

## Value-creation step

!!! note "Your paths"
    This step is written for the Product path. On other paths, use this time for your next milestone in [Choose your paths](../paths.md).

Put the capstone in front of your ~10 users for real use this week, and measure **retention**: who came back.

- Define an active user in one sentence (for example, "completed at least one task").
- Make a small table: each user, the days they were active, and whether they returned after their first week.
- Talk to one user who did not come back. Ask what they did instead.
- Follow up on the paid-pilot or letter-of-intent proposals from Week 11.
