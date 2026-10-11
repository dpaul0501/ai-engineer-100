# Week 1 (Days 1–7): Lookups, Vectors, and Your First LLM Call

## This week

- **You will learn:**
    - **DSA:** how hashing turns search into lookup.
    - **ML/DL:** how NumPy represents data as arrays and how vectors measure similarity.
    - **AI agents:** what an LLM API call actually sends and returns.
    - **Software dev:** how git, pytest, and seeded bugs prove that code works.
- **Goal:** by Sunday you can build a small, tested search tool from scratch and defend every line of it.
- **Weekly build:** a pocket search CLI over a folder of markdown files, with an LLM answer on top. See [Weekly build](#weekly-build).
- **AI rule (Days 1–20):** AI explains, you type. Ask AI to explain or review. Do not paste code it writes.

## DSA: arrays and hashing

**Read first:** [Hashing](https://mldsa.github.io/docs/part1/chapter01-hashing/) and [Top-K](https://mldsa.github.io/docs/part1/chapter02-topk/) in *From LeetCode to Learning Systems*.

**Focus.** A hash table answers "have I seen this?" in O(1) average time. Most problems this week follow one pattern: scanning a list costs O(n) per question, so you store what you have seen and look it up instead. The cost is memory, and the worst case is bad when keys collide.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 1 | [1. Two Sum](https://leetcode.com/problems/two-sum/) | Look up the complement |
| 1 | [217. Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | Set membership |
| 2 | [242. Valid Anagram](https://leetcode.com/problems/valid-anagram/) | Count, then compare |
| 2 | [49. Group Anagrams](https://leetcode.com/problems/group-anagrams/) | Canonical key |
| 3 | [347. Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Count, then select |
| 4 | [238. Product of Array Except Self](https://leetcode.com/problems/product-of-array-except-self/) | Prefix and suffix scans |
| 5 | [128. Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | Start only at run beginnings |
| 5 | [36. Valid Sudoku](https://leetcode.com/problems/valid-sudoku/) | One set per constraint |

### Depth ladder: Two Sum

1. **Solve.** One pass with a dict from value to index. O(n) time, O(n) space. Write tests for the given examples first.
2. **Vary.**
    - The input is sorted. Two pointers give O(1) extra space. When is that better than the dict?
    - Return *all* pairs, not one. How do you avoid reporting the same pair twice?
    - Numbers arrive as a stream and queries interleave with inserts (LeetCode 170, Two Sum III). Do you optimize `add` or `find`?
    - Three Sum and k-Sum. How does the cost grow with k?
3. **Break the assumptions.**
    - `target = 2 * x` with `x` appearing once. Does your code wrongly pair `x` with itself?
    - **Floats.** `0.1 + 0.2 == 0.3` is `False`. A dict lookup needs exact equality, so the complement lookup misses. What do you do when you need "close to target"?
    - **Bigger than memory.** A billion numbers on disk. Partition them by hash into files that fit in memory, then solve each file.
    - **Collisions.** In CPython, `hash(-1) == hash(-2) == -2`. Run that. Then read about why Python randomizes string hashes (`PYTHONHASHSEED`): crafted keys that all collide turn O(1) lookups into O(n).
4. **ML lens.**
    - When "close to target" replaces "equal to target", the problem becomes nearest-neighbor search. Approximate nearest-neighbor search (LSH, FAISS) powers retrieval in RAG systems in Week 6.
    - **The hashing trick.** scikit-learn's `FeatureHasher` and `HashingVectorizer` map features to column indices by hashing, so they never store a vocabulary. What do collisions cost the model?
5. **In the wild.**
    - CPython's dict: read `Objects/dictobject.c` and the design notes in `Objects/dictnotes.txt`. How does it probe after a collision? Why does a dict keep insertion order?
    - PostgreSQL hash join: `src/backend/executor/nodeHashjoin.c`. It is Two Sum with `=` in place of `+`. Build a table from one side, probe it with the other. What happens when the table does not fit in `work_mem`?
    - pandas `merge` and `groupby` use the hash tables in `pandas/_libs/hashtable`.

### Depth ladder: Contains Duplicate

- **Vary:** within distance k (LeetCode 219); within value t and distance k (LeetCode 220).
- **Break:** you have 10 billion URLs and a few hundred MB of RAM. An exact set does not fit.
- **ML lens:** duplicates shared between train and test splits leak labels. Write a check that fails if any row hash appears in both splits.
- **In the wild:** Bloom filters answer "definitely not seen" in a fixed amount of memory. RocksDB and Cassandra use them to skip files that cannot contain a key. Implement one in 30 lines and measure its false-positive rate against the formula.

### Depth ladder: Group Anagrams

- **Vary:** key by `sorted(s)` at O(k log k) per word, or by a 26-count tuple at O(k). Which wins for long words, and which for short ones?
- **Break:** `"é"` can be one code point or two (`e` plus a combining accent). Then `sorted()` disagrees on whether two strings are anagrams. Fix it with `unicodedata.normalize("NFC", s)`. Uppercase letters break it too.
- **ML lens:** a canonical key turns grouping into a lookup. Pretraining pipelines dedupe text the same way: exact hashes catch identical documents, and MinHash catches near-duplicates.
- **In the wild:** read `MinHashLSH` in the `datasketch` library. How do bands and rows trade precision for recall?

### Depth ladder: Top K Frequent Elements

- **Vary:** a heap gives O(n log k); bucket sort by count gives O(n); `np.argpartition` gives O(n) on arrays. Benchmark all three at n = 10⁶.
- **Break:** ties at the k-th place. Is your output deterministic? The data is an endless stream: look up Misra–Gries and Count-Min Sketch.
- **ML lens:** **top-k sampling** in LLM decoding keeps the k highest logits and renormalizes. Tokenizers keep the most frequent tokens or merges when building a vocabulary ([tokenization chapter](https://dpaul0501.github.io/llm-agent-reads/part_01_embeddings/)).
- **In the wild:**
    - `collections.Counter.most_common(n)` in `Lib/collections/__init__.py` calls `heapq.nlargest`. Why that, and not a full sort?
    - Hugging Face `transformers`: `TopKLogitsWarper` in `generation/logits_process.py`. What value does it write into the logits it removes, and why that value?

### Depth ladder: Product of Array Except Self

- **Vary:** solve it without division in O(1) extra space, not counting the output.
- **Break:**
    - One zero, then two zeros. The division shortcut fails.
    - Products of 10⁵ numbers overflow int64 in NumPy, and underflow to 0.0 as floats. Fix it by working in log-space: sum logs instead of multiplying.
- **ML lens:** prefix and suffix scans appear everywhere.
    - Top-p (nucleus) sampling sorts probabilities and takes a cumulative sum to find the cutoff (`TopPLogitsWarper`, same file as above).
    - Causal attention gives each token only its prefix ([attention chapter](https://dpaul0501.github.io/llm-agent-reads/part_03_attention/)).
    - State-space models such as Mamba compute their recurrence with a parallel scan.
- **In the wild:** `np.cumsum` and `torch.cumsum`. Why is a parallel prefix sum possible at all? Look up the term "associative operator".

### Day 5 extras: Longest Consecutive Sequence, Valid Sudoku

- **Longest Consecutive Sequence.** The ML lens is finding runs in event logs: user sessions, sensor dropouts. Write the pandas version with `s.diff().ne(1).cumsum()` and compare it with your set version.
- **Valid Sudoku.** One set per row, column, and box is a constraint check. The ML lens is data validation: uniqueness and range checks before training. Write three such checks for a CSV.

## ML/DL: NumPy and vectors

### Learn

- Arrays, `dtype`, `shape`, and the difference between views and copies
- Broadcasting rules, and `axis` and `keepdims`
- Vectorization: replace Python loops with array operations
- Dot product, L2 norm, cosine similarity, Euclidean distance
- The math in the [foundation chapter](https://dpaul0501.github.io/llm-agent-reads/module_01_foundations/)

### Read the code

**1. This repo's [`math_utils.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/math_utils.py) and [`self_attention.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/self_attention.py).** Answer in your log:

- `softmax` subtracts the max before `exp`. Why? Run `np.exp(1000.0)`.
- `softmax` casts its input to float64. Pass it a float32 array and check the output dtype. What does that cast cost in memory, and when would a library refuse to do it?
- `scaled_dot_product` calls `k.transpose(0, 2, 1)`. Which input shapes does that support? Try a 4-D (batch, heads, seq, dim) tensor.
- The mask fills blocked scores with `-1e9`. Mask out an entire row. Does the result become NaN, or something worse: a silent, plausible-looking average? What would `-np.inf` do?

**2. scikit-learn's `euclidean_distances`** in `sklearn/metrics/pairwise.py`. It computes `‖x‖² − 2x·y + ‖y‖²` instead of `‖x − y‖²`.

- Why is that faster? Because it becomes one matrix multiply.
- Why is it less precise? Find where the code clips negative results to zero, and the special path for float32 input.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `softmax` (llm-agent-reads) | `axis` | `-1` | Which axis sums to 1 | Wrong axis: rows no longer sum to 1, and shape tests still pass |
| `np.linalg.norm` | `ord` | `None` (L2) | Which norm | L1 vs L2 vs ∞ rank neighbors differently |
| `np.linalg.norm` | `axis`, `keepdims` | `None`, `False` | Per-row vs whole-array norm | Without `keepdims`, dividing by the norm broadcasts wrongly or errors |
| `euclidean_distances` | `squared` | `False` | Skip the square root | Ranking is unchanged, but values are not distances |
| `euclidean_distances` | `X_norm_squared` | `None` | Reuse precomputed norms | Stale norms return silently wrong distances |
| `cosine_similarity` | `dense_output` | `True` | Sparse vs dense result | A dense n×n result on 10⁶ rows exhausts memory |

### Break it

Find two vectors where cosine similarity and Euclidean distance disagree about which neighbor is closest. Then normalize both vectors and show that the two now agree. Explain why.

## AI agents: the LLM API call

### Learn

- Messages and roles: system, user, assistant
- Sampling parameters: temperature, max output tokens, stop sequences
- The response: content, **stop reason**, and token usage (input and output)
- Errors: rate limits, timeouts, retries with backoff

### Experiment

Send the same prompt 20 times at temperature 0, and 20 times at temperature 1. Count distinct answers. Log input and output tokens, and compute the cost of all 40 calls using your provider's current price page.

### Break it

- Set max output tokens to 10. What is the stop reason? Does your code notice, or does it use the truncated answer as if it were complete?
- Ask for JSON, then parse it. How often does parsing fail? Week 2 fixes this with structured output.
- Temperature 0 is often not bit-for-bit deterministic. Did your 20 runs agree exactly?

### ML lens

Temperature divides the logits before softmax. That is the same `softmax` you read above. See the temperature plot in the [inference chapter](https://dpaul0501.github.io/llm-agent-reads/part_06_inference_and_tricks/).

### In the wild

Open your provider's official Python SDK and find its retry logic (search for `max_retries`). Which errors does it retry? How long does it wait between attempts?

## Software dev: git, pytest, and proof

### Learn

- A repo per learner: `venv`, `requirements.txt`, small commits with messages that say why
- pytest basics: `assert`, `parametrize`, fixtures
- Type hints on every public function

### Apply it: seeded bugs

Clone [llm-agent-reads](https://github.com/dpaul0501/llm-agent-reads). Break [`softmax`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/math_utils.py) by deleting the max subtraction, then run `pytest`. **The test suite still passes.** `test_softmax_basic` uses `[1, 2, 3]`, where `exp` never overflows.

Write a test that fails on the broken version. Then do the same for two more seeded bugs:

- Ignore `mask` in `scaled_dot_product_attention`. `test_self_attention_shape` checks only shapes.
- Change `softmax`'s default `axis` to `0`.

### Verify

A test proves something only if it fails when the code is wrong. For every test you add this week, record the seeded bug it catches.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 1 | Two Sum, Contains Duplicate | Arrays, dtype, broadcasting | First API call; roles | Repo, venv, first commit |
| 2 | Valid Anagram, Group Anagrams | Dot product, norms | Temperature experiment | pytest, parametrize |
| 3 | Top K Frequent | Cosine vs Euclidean; read `euclidean_distances` | Tokens, usage, stop reason | Type hints, fixtures |
| 4 | Product Except Self | Read `softmax` and attention masking | Errors, retries, timeouts | Seeded bugs in llm-agent-reads tests |
| 5 | Longest Consecutive, Valid Sudoku | Vectorize a loop and benchmark it | Prompt that answers from given snippets | Code review checklist |
| 6 | **Weekly build** | | | |
| 7 | **Defend and publish** | | | |

## Weekly build

### Pocket search CLI

```bash
pocket index ./docs          # build an index of a folder of .md files
pocket search "rotary position" -k 5
pocket ask "why does softmax subtract the max?"
```

**Required behavior**

- **Index (DSA + ML/DL).**
    - Tokenize to lowercase words.
    - Build the vocabulary with a dict.
    - Represent each document as a TF-IDF vector in NumPy.
    - Save the index to disk.
- **Search (DSA + ML/DL).**
    - Cosine similarity between the query and every document, vectorized with no Python loop over documents.
    - Top-k by `argpartition` or a heap.
- **Ask (AI agents).**
    - Send the top 3 snippets and the question to an LLM.
    - Instruct it to answer only from the snippets and to cite file names.
    - Check the stop reason, and handle API errors.
- **Engineering (Software dev).**
    - Type hints throughout.
    - pytest suite.
    - A README with setup steps and an example run.

Use the `docs/` folder of [LLM Foundations from Scratch](https://github.com/dpaul0501/llm-agent-reads) as the first corpus. The starter code, acceptance tests, and defense questions are in the bootcamp labs repo.

**Stretch**

- Benchmark indexing and search time at 1k, 10k, and 100k documents, and plot both.
- Swap TF-IDF for the hashing trick. Measure memory saved and ranking quality lost.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 1 | "Two Sum is a database hash join." Walk through PostgreSQL's `nodeHashjoin.c`. |
| 2 | "Why `sorted(s)` says two strings are not anagrams when they are." Unicode normalization. |
| 3 | "Why scikit-learn's Euclidean distance can go negative before it clips." |
| 4 | "I broke softmax and the tests still passed." Show the seeded bug and the test that catches it. |
| 5 | "I sent the same prompt 40 times. Here is what temperature actually did." |
| 6 | Your pocket search tool, running on a real folder, with a screenshot. |
| 7 | What the defense exposed, and what you fixed. |

**This week's write-up:** a GitHub README for your pocket search CLI that a stranger can install and run.

## Value-creation step

Write down three problems you have seen people around you struggle with, where search, data, or an AI assistant could help. For each one, name one person you could ask about it. You will interview them in Week 4.
