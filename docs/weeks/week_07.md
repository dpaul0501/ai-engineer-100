# Week 7 (Days 43–49): Graphs, Tokens, and Judges

## This week

- **You will learn:**
    - **DSA:** how BFS and DFS explore a graph, and how topological order schedules work that depends on other work.
    - **ML/DL:** how byte-pair encoding turns text into tokens, and how an embedding table turns tokens into vectors.
    - **AI agents:** how to evaluate an agent, and when an LLM judge can be trusted to grade one.
    - **Software dev:** how to review a pull request so that it catches real bugs, and how to write one that is easy to review.
- **Goal:** by Sunday you can train a tokenizer on your own corpus, explain every merge it made, train embeddings on its tokens, and say with a number how much your LLM judge agrees with you.
- **Weekly build:** a BPE tokenizer and embeddings lab over your Week 6 corpus, graded by a calibrated LLM judge, delivered as a reviewed pull request. See [Weekly build](#weekly-build).
- **AI rule (Days 21–60):** AI pairs, you prove. AI may write code. Tests come first, and you must explain every line it wrote.

## DSA: graphs (BFS and DFS)

**Focus.** A graph is nodes plus edges, stored as an adjacency list (a dict from node to neighbors). Two traversals cover most problems. **BFS** uses a queue and visits nodes in order of distance, so it finds shortest paths when every edge costs the same. **DFS** uses a stack (or recursion) and goes deep first, which suits connectivity, cycle detection, and ordering. Both are O(V + E). Both need a `visited` set, or a cycle makes them loop forever. Many grid problems are graph problems in disguise: each cell is a node, and its four neighbors are its edges.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 43 | [200. Number of Islands](https://leetcode.com/problems/number-of-islands/) | Count connected components |
| 43 | [695. Max Area of Island](https://leetcode.com/problems/max-area-of-island/) | Component size by DFS |
| 44 | [133. Clone Graph](https://leetcode.com/problems/clone-graph/) | Map old node to new node |
| 45 | [994. Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) | Multi-source BFS by levels |
| 46 | [207. Course Schedule](https://leetcode.com/problems/course-schedule/) | Cycle detection in a directed graph |
| 46 | [417. Pacific Atlantic Water Flow](https://leetcode.com/problems/pacific-atlantic-water-flow/) | Search backward from the targets |
| 47 | [127. Word Ladder](https://leetcode.com/problems/word-ladder/) | BFS shortest path over implicit edges |

### Depth ladder: Number of Islands

1. **Solve.** Scan the grid. At each unvisited `"1"`, start a DFS that marks the whole island, and add one to the count. O(rows × cols) time. Write tests first: an empty grid, all water, all land, and a diagonal pair (not connected).
2. **Vary.**
    - Use BFS instead of DFS. Which uses less memory on a long snake-shaped island?
    - Use union-find. When does union-find win? (Hint: land cells added one at a time, LeetCode 305.)
    - Count islands with 8-way connectivity. Which line changes?
3. **Break the assumptions.**
    - A 1,000 × 1,000 grid of all land. Recursive DFS goes 10⁶ deep. What happens in Python, and how does an explicit stack fix it?
    - Your solution overwrites `"1"` with `"0"` to mark visited. The caller's grid is now destroyed. Is that acceptable? Write a test that catches it.
    - The grid is too big for memory and arrives row by row. Can you count islands keeping only two rows? (Union-find across rows.)
4. **ML lens.** **Connected-component labeling** in images: segment a mask into separate objects, then count or measure them. In data cleaning, entity resolution builds a graph of "these two records match" and takes connected components as the merged entities.
5. **In the wild.**
    - SciPy's `scipy.ndimage.label` in `scipy/ndimage/_measurements.py`. Its `structure` argument sets the connectivity. What is the default, and how do you get 8-way connectivity in 2-D?
    - NetworkX `connected_components`. Compare its traversal with yours, then read `bfs_edges` in `networkx/algorithms/traversal/breadth_first_search.py`.

### Depth ladder: Course Schedule

1. **Solve.** Build the graph from prerequisites. Either DFS with three colors (unvisited, in progress, done), where reaching an in-progress node means a cycle, or Kahn's algorithm: repeatedly take nodes with in-degree 0. If some nodes are never taken, there is a cycle. O(V + E).
2. **Vary.**
    - Return an order (LeetCode 210, in Week 12). With Kahn's algorithm the order falls out. With DFS, it is the reverse of finish order.
    - Return the cycle itself, not just "impossible". Which version makes that easy?
    - Courses that can run in parallel: the minimum number of semesters is the number of BFS levels in Kahn's algorithm.
3. **Break the assumptions.**
    - A course that is its own prerequisite. Duplicate edges. Courses that appear in no edge. Test all three.
    - Several valid orders exist. Is your output deterministic? Should it be?
4. **ML lens.**
    - **Autograd.** A computation graph is a DAG. The backward pass must visit each node only after every node that uses it. You built this in Week 3, probably with a DFS topological sort.
    - **BPE merges.** Each merged token is built from two earlier tokens. The merge table is a DAG, and decoding a token is a DFS down to bytes. This week's build uses that.
5. **In the wild.**
    - CPython's `Lib/graphlib.py`: `TopologicalSorter`. Read `prepare`, `get_ready`, and `done`. How does it support running ready nodes in parallel? What does it raise on a cycle?
    - PyTorch's autograd engine: `Engine::compute_dependencies` in `torch/csrc/autograd/engine.cpp`. It counts, for each node, how many inputs must finish before it can run. That is the in-degree from Kahn's algorithm.

### Depth ladder: Word Ladder

1. **Solve.** BFS from the start word. Neighbors are words in the list that differ by one letter. Return the level at which you reach the end word. Precompute neighbors with wildcard keys: `"hot"` maps to `"*ot"`, `"h*t"`, `"ho*"`.
2. **Vary.**
    - Bidirectional BFS: search from both ends and stop when the frontiers meet. Measure the nodes visited both ways on a 5,000-word list.
    - Return every shortest ladder (LeetCode 126). Why does this need the BFS parent map, not just the distance?
3. **Break the assumptions.**
    - The end word is not in the list. Words of different lengths. A list of 10⁵ words: comparing every pair is 10¹⁰ checks. The wildcard keys fix that.
4. **ML lens.** The wildcard key is a canonical key, the same trick as Group Anagrams in Week 1. Grouping by a shared key to avoid comparing all pairs is how locality-sensitive hashing finds near-duplicate text.
5. **In the wild.** NetworkX `bidirectional_shortest_path` in `networkx/algorithms/shortest_paths/unweighted.py`. Which frontier does it expand at each step, and why?

### Shorter ladders: 695, 133, 994, 417

- **Max Area of Island.** **Vary:** return the size of every island and plot the histogram. **ML lens:** `scipy.ndimage.label` followed by a count per label is the same thing.
- **Clone Graph.** **Break:** a node that lists itself as a neighbor. Without the old-to-new map you recurse forever. **In the wild:** `copy.deepcopy` in `Lib/copy.py` takes a `memo` dict for exactly this reason. Deep-copy a list that contains itself.
- **Rotting Oranges.** **Vary:** start BFS from every rotten orange at once (multi-source). **ML lens:** label propagation spreads known labels to unlabeled neighbors in a similarity graph. scikit-learn's `LabelPropagation` is in `sklearn/semi_supervised/_label_propagation.py`.
- **Pacific Atlantic Water Flow.** **Vary:** instead of searching from every cell, run one multi-source BFS inward from each ocean and intersect the results. **Break:** equal neighboring heights let water flow both ways, so the comparison must be `>=`, not `>`. **ML lens:** the watershed algorithm for image segmentation floods outward from markers in the same way; see `skimage.segmentation.watershed`.

## ML/DL: tokenization and embeddings

### Learn

Read the [tokenization and embeddings chapter](https://dpaul0501.github.io/llm-agent-reads/part_01_embeddings/). Then:

- **Why subwords.** Characters make sequences long. Whole words make the vocabulary huge and leave unknown words. Subwords sit between.
- **Byte-pair encoding (BPE)** (Sennrich et al., 2016, [arXiv:1508.07909](https://arxiv.org/abs/1508.07909)). Start with single symbols. Repeatedly merge the most frequent adjacent pair into a new token. The order of merges is the tokenizer.
- Try merges by hand in the interactive [BPE lab](https://dpaul0501.github.io/llm-agent-reads/assets/lab_bpe.html) before you write code.
- **Byte-level BPE.** Start from the 256 byte values instead of characters, so any text can be encoded and nothing is unknown.
- **Pre-tokenization.** A regex splits text into pieces (words, numbers, punctuation) before BPE runs. Merges never cross those boundaries.
- **Special tokens** such as end-of-text are added by the program, not learned. User text that contains their literal string must not turn into them.
- **Vocabulary size trade-off.** A larger vocabulary gives shorter sequences but a bigger embedding table and rarer tokens with less training signal.
- **Embeddings.** A token ID is a row index into a learned matrix (section 3 of the chapter). Skip-gram learns rows so that tokens appearing in similar contexts get similar vectors (section 8).

### Read the code

**1. This repo's [`embedding.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/embedding.py).** It is 15 lines. Answer in your log:

- The weights are drawn with standard deviation `1 / sqrt(d_model)`. What is the expected L2 norm of one row? Check it numerically.
- Call it with token ID `vocab_size`. Then with `-1`. One raises an error. The other returns a row silently. Which row, and why is that worse?
- Call it with `[2.7]`. The cast to `int64` truncates. Which row comes back?
- `test_token_embedding_shape` in `tests/test_foundations.py` checks only the shape. Write a test that fails if `-1` is accepted.

**2. tiktoken's teaching implementation**, [`tiktoken/_educational.py`](https://github.com/openai/tiktoken/blob/main/tiktoken/_educational.py).

- In `bpe_train`, how is the most common pair chosen? Two pairs tie. Which one wins, and is that deterministic across Python runs?
- `bpe_train` loops `while len(ranks) < vocab_size`. Train on the text `"ab"` with `vocab_size=300`. What happens when no pairs are left?
- `bpe_encode` scans every pair on every merge. What is its complexity on a piece of length n? Compare with `_byte_pair_merge_large` in `src/lib.rs` (Week 6).

**3. tiktoken's `Encoding.encode`** in `tiktoken/core.py`. By default it raises an error if the text contains a special token's literal string. Why is that the safe default for text that came from a user?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `tiktoken` `Encoding.encode` | `allowed_special` | `set()` | Special-token strings that may become special tokens | `"all"` lets user text inject end-of-text or other control tokens |
| `tiktoken` `Encoding.encode` | `disallowed_special` | `"all"` | Special strings that raise an error | `()` encodes them as ordinary text silently, which may be what you want for logs |
| `tiktoken` `_educational.bpe_train` | `vocab_size` | none (required, at least 256) | Number of tokens: 256 bytes plus merges | Below 256 raises; above the number of possible merges fails on an empty count |
| `tiktoken` `_educational.bpe_train` | `visualise` | `"colour"` | Prints each merge | Left on, training on a real corpus floods the terminal and slows down |
| Hugging Face `BpeTrainer` | `vocab_size` | `30000` | Target vocabulary size | Too large for a small corpus gives rare tokens that are barely trained |
| Hugging Face `BpeTrainer` | `min_frequency` | `0` | Minimum count for a merge | 0 merges pairs seen once, which memorizes typos |
| `nn.Embedding` | `padding_idx` | `None` | A row that gets no gradient | Without it, padding tokens get trained and leak into averages |
| `nn.Embedding` | `max_norm` | `None` | Renormalizes looked-up rows | Set, it modifies the weight tensor in place during forward |
| `nn.Embedding` | `sparse` | `False` | Sparse gradients for the table | `True` needs an optimizer that supports sparse gradients, such as `SparseAdam` |

### Break it

Train your BPE on English text, then encode Hindi, Japanese, and Python code. Count tokens per character for each. The same sentence can cost several times more tokens in one language than another. Then encode a string that ends in the middle of a multi-byte UTF-8 character and decode it. What does your decoder do with the incomplete bytes?

## AI agents: agent evals and LLM-as-judge

### Learn

- **What to evaluate.** The final answer, and the path: did the agent call the right tools, with the right arguments, in a sensible number of steps?
- **Three kinds of check:**
    - **Code checks:** exact match, a JSON schema, a regex, "the cited chunk was retrieved". Cheap and reliable. Use them first.
    - **Reference-based:** compare with a known good answer.
    - **Rubric-graded:** a judge scores against written criteria when no exact answer exists.
- **Repeat runs.** Agents are not deterministic. Run each case several times and report the pass rate, not one pass.
- **LLM-as-judge.** Pointwise (score one answer) or pairwise (which of two is better). Give the judge a rubric, ask for a short reason before the verdict, and ask for a fixed output format.
- **Known judge biases** (Zheng et al., 2023, [arXiv:2306.05685](https://arxiv.org/abs/2306.05685)):
    - **Position bias:** preferring the first (or second) answer shown.
    - **Verbosity bias:** preferring longer answers.
    - **Self-preference:** preferring answers written in its own style.
- **Calibration.** A judge is only useful if it agrees with you. Label a sample yourself, then measure agreement with Cohen's kappa, which corrects for agreement by chance.

### Experiment

Take 50 answers from your Week 6 RAG system. Label each one yourself as *supported* or *not supported* by its cited chunks, before you run any judge. Then:

- Run a pointwise judge with a rubric. Compute accuracy, precision and recall for *not supported*, and Cohen's kappa against your labels.
- Run a pairwise judge on 20 pairs, once in each order. Count how often the verdict flips when only the order changes. That is your position-bias rate.
- Change one thing in the judge prompt (add examples, or require a quote from the chunk as evidence). Rerun. Did kappa go up?

### Break it

- Take a wrong answer and pad it with confident, relevant-sounding sentences. Does the judge's score go up?
- Put the expected verdict in the prompt by accident (for example, a file name like `bad_answer_07`). Does the judge follow it?
- Ask the judge to grade an answer it cannot check (the chunk is missing). Does it say so, or guess?

### ML lens

A judge is a classifier, and your labels are the test set. Precision, recall, and the confusion matrix from Week 4 apply directly. If only 10% of answers are unsupported, a judge that always says "supported" scores 90% accuracy and kappa 0. That is why you report kappa.

### In the wild

- scikit-learn's `cohen_kappa_score` in `sklearn/metrics/_classification.py`. Read the formula in the code. What does the `weights` argument add for ordinal scores like 1–5?
- The MT-Bench paper above reports agreement between strong LLM judges and human experts, and how swapping positions changes verdicts. Compare their position-bias rate with yours.

## Software dev: PR review

### Learn

- **A reviewable PR:** small (a few hundred lines at most), one purpose, a description that says what changed, why, and how it was tested.
- **Review order:** read the description, then the tests, then the public interface, then the implementation. Tests tell you what the author thinks the code does.
- **Check out and run it:** `gh pr checkout <number>`, then run the tests and one manual case. Reading alone misses things.
- **Comment kinds:** mark each comment *blocking*, *question*, or *nit*. A blocking comment names the line, the input that breaks it, and the test that would catch it.
- **Reviewing AI-written code:** look for plausible but wrong edge cases, unused parameters, silently swallowed errors, and tests that only check shapes or types.
- **As the author:** answer every comment, push a commit per fix, and do not resolve a blocking comment without a test.

### Apply it

- Pair with another participant. Each of you plants one bug in your Week 6 RAG repo, opens a pull request that looks like a normal change, and asks the other to review it.
- Then open this week's build as a pull request, and have the same partner review it.

### Verify

- Did your reviewer find the planted bug? If not, which part of the review process would have found it?
- Every blocking comment on your build PR is closed by a commit that adds or changes a test. Link each one in your log.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 43 | Number of Islands, Max Area of Island | Tokenization chapter; subwords and BPE | What to evaluate; code checks first | What makes a PR reviewable |
| 44 | Clone Graph | Read `_educational.py`; train BPE on a toy text | Label 50 Week 6 answers yourself | Review order; `gh pr checkout` |
| 45 | Rotting Oranges | Byte-level BPE; regex pre-split | Pointwise judge; kappa against your labels | Plant a bug; open the PR for your partner |
| 46 | Course Schedule, Pacific Atlantic | Read `embedding.py`; seeded-bug test | Pairwise judge; position-bias rate | Review your partner's PR |
| 47 | Word Ladder | Skip-gram embeddings in PyTorch | Break it: padding, leaked labels, missing evidence | Respond to review; one commit per fix |
| 48 | **Weekly build** | | | |
| 49 | **Defend and publish** | | | |

## Weekly build

### BPE tokenizer and embeddings lab

```bash
tok train ./corpus --vocab-size 2000 --out merges.json
tok encode "patients missed 3 appointments"
tok compare ./corpus                        # tokens per word vs a tiktoken encoding
emb train --dim 64 --window 2               # skip-gram on your tokens
emb neighbors "appointment" -k 10
emb judge --labels labels.csv               # LLM judge, calibrated against your labels
```

Use your Week 6 corpus, so the tokens and neighbors come from your problem area.

**Required behavior**

- **Tokenizer (ML/DL).**
    - Byte-level BPE trained from scratch, with a regex pre-split.
    - `decode(encode(s)) == s` for any valid UTF-8 string, including emoji and non-Latin scripts.
    - Special tokens are added explicitly. Their literal strings in user text raise an error by default, as in tiktoken.
    - Merges saved to a file and loaded back with identical output.
- **Merge graph (DSA).**
    - Load the merges as a DAG. Validate them with a topological sort: every merge's two parts must exist before it.
    - A corrupted merges file with a cycle or a missing part is rejected with a clear error.
    - Expand any token to its bytes by DFS with memoization.
- **Comparison (ML/DL).**
    - Tokens per word on your corpus for your tokenizer and for one tiktoken encoding. Explain the gap.
- **Embeddings (ML/DL).**
    - Skip-gram with negative sampling in PyTorch, using `nn.Embedding`, trained on your tokens.
    - Nearest neighbors by cosine for 10 query tokens you choose from your problem area.
- **Judge (AI agents).**
    - Label 50 (query, neighbor) pairs yourself as related or not, before running the judge.
    - An LLM judge with a written rubric labels the same pairs.
    - Report accuracy, Cohen's kappa, and the flip rate when the neighbor list order is reversed.
- **Review (Software dev).**
    - Delivered as a pull request. A partner reviews it, and every blocking comment is closed by a commit with a test.

**Stretch**

- Speed up training with a heap of pair counts that updates incrementally after each merge, instead of recounting. Benchmark both.
- Build a k-nearest-neighbor graph over your token embeddings and find its connected components with BFS. Do the components match topics in your problem area?

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 43 | "`scipy.ndimage.label` is Number of Islands." Show the default connectivity and how to change it. |
| 44 | "I asked BPE for 300 tokens on a 2-byte text." Show what `bpe_train` does when it runs out of pairs. |
| 45 | "My LLM judge agreed with me N% of the time. Kappa told a different story." Show the confusion matrix with your real numbers. |
| 46 | "Embedding lookup with token ID -1 returns a row, not an error." Show the test that catches it. |
| 47 | "Swapping the order of two answers flipped the judge's verdict in N of 20 pairs." |
| 48 | Your tokenizer on your own corpus: the first 20 merges, and what they say about your problem area. |
| 49 | What the defense and the PR review exposed, and what you fixed. |

**This week's write-up:** a **GitHub** README for the tokenizer and embeddings lab that a stranger can reproduce: train, encode, compare, and the judge's agreement numbers.

## Value-creation step

Recruit **10 people who agree to try your MVP by Day 60**. Start with the interviewees and the two people who saw your sketch in Week 6, then ask each for one more name. Send each person the one-page sketch and a direct question: "Will you try this for a week before Day 60?" Count only a clear yes. Track them in a small SQL table (Week 5): name, the pain they have, how you reached them, and the date they agreed.
