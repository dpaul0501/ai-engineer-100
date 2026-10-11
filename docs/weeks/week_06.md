# Week 6 (Days 36–42): Heaps, Neural Nets, and Retrieval

## This week

- **You will learn:**
    - **DSA:** how a heap keeps the best k items in O(log k) per item, and merges sorted streams.
    - **ML/DL:** how a neural network is a stack of layers trained by backprop, and how PyTorch runs that loop.
    - **AI agents:** how retrieval-augmented generation (RAG) finds the right text and how to measure whether it did.
    - **Software dev:** how Docker makes a build reproducible and how CI blocks a change that breaks it.
- **Goal:** by Sunday you can answer questions over documents from your own problem area, prove with numbers how often retrieval finds the right chunk, and ship it as a container that CI tests on every pull request.
- **Weekly build:** a RAG system over your problem area, with a hybrid retriever, a small PyTorch reranker, and an eval set. See [Weekly build](#weekly-build).
- **AI rule (Days 21–60):** AI pairs, you prove. AI may write code. Tests come first, and you must explain every line it wrote.

## DSA: heaps

**Read first:** [Top-K](https://mldsa.github.io/docs/part1/chapter02-topk/) in *From LeetCode to Learning Systems*.

**Focus.** A heap is a binary tree stored in an array, where each parent is no larger (min-heap) than its children. You get the smallest item in O(1) and push or pop in O(log n). Reach for it when you need "the best k so far" or "the next smallest across many sources", and you do not need the rest in order. To keep the k *largest*, keep a *min*-heap of size k: the root is the weakest survivor, so each new item is compared with it once.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 36 | [703. Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) | Min-heap of size k |
| 36 | [1046. Last Stone Weight](https://leetcode.com/problems/last-stone-weight/) | Max-heap by negation |
| 37 | [215. Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | Heap vs quickselect |
| 38 | [973. K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | Top-k by distance |
| 39 | [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | Heap of list heads |
| 40 | [621. Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Greedy by count, with cooldown |
| 40 | [295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Two heaps, balanced |

### Depth ladder: K Closest Points to Origin

1. **Solve.** Keep a max-heap of size k on distance (push negated squared distance). O(n log k) time, O(k) space. Compare squared distances: the square root does not change the order. Write tests first, including ties.
2. **Vary.**
    - Closest to an arbitrary query point, not the origin. What changes?
    - Points arrive as a stream and queries interleave. Which structure do you keep?
    - `np.argpartition` on the distance array, then sort only the k winners. Benchmark it against the heap at n = 10⁶, k = 10.
    - Cosine similarity instead of distance. Why does normalizing the vectors first make the two give the same order (Week 1)?
3. **Break the assumptions.**
    - **Ties.** Many points at the same distance. Is your output deterministic? Add the index as a tiebreaker in the heap tuple and test that two runs agree.
    - **Overflow.** Coordinates near 10⁹ in int32 NumPy arrays. `x*x` overflows silently. Show it, then fix the dtype.
    - **Too many points for memory.** Read the file in chunks, keep a heap of size k across chunks. Prove the answer matches the in-memory version.
4. **ML lens.** This is exactly dense retrieval. The "points" are chunk embeddings, the "origin" is the query embedding, and k is how many chunks go into the prompt. Exact search is this loop. Approximate search (HNSW, IVF) trades a little recall for speed when n is in the millions.
5. **In the wild.**
    - FAISS: `faiss/IndexFlat.cpp` calls `knn_L2sqr` and `knn_inner_product` in `faiss/utils/distances.cpp`, which keep a heap of size k per query using `faiss/utils/Heap.h`. Find `heap_replace_top`. Why replace the top instead of pop then push?
    - CPython: `nsmallest` and `nlargest` in `Lib/heapq.py`. For small k they keep a heap. When do they fall back to `sorted`? Read the comments with the comparison counts.

### Depth ladder: Merge k Sorted Lists

1. **Solve.** Push the head of each list into a min-heap as `(value, list_index, node)`. Pop the smallest, push its successor. O(N log k) for N total nodes. Why does the tuple need `list_index`?
2. **Vary.**
    - Divide and conquer: merge pairs of lists in rounds. Same O(N log k). Which has better constants in Python?
    - The inputs are files too large for memory. This is external merge sort.
    - Merge with deduplication: equal values from different lists appear once.
3. **Break the assumptions.**
    - Two nodes with equal values and no tiebreaker: Python tries to compare `ListNode` objects and raises `TypeError`. Reproduce it.
    - Some lists are empty, or all are. `k = 0`.
    - One list has 10⁶ nodes and the rest have 1. Is the heap still worth it?
4. **ML lens.** **Hybrid retrieval** merges ranked lists: one from keyword search (BM25), one from embeddings. Reciprocal rank fusion scores each chunk by `Σ 1 / (60 + rank)` across lists. Sharded vector search merges the top-k from each shard the same way.
5. **In the wild.**
    - CPython: `heapq.merge` in `Lib/heapq.py`. It merges any number of sorted iterables lazily. How does it handle `key` and `reverse`?
    - RocksDB: `table/merging_iterator.cc`. Reads in an LSM tree merge sorted runs from many files with a heap. Read the header comment.

### Depth ladder: Find Median from Data Stream

1. **Solve.** A max-heap for the lower half, a min-heap for the upper half. Keep sizes within one. `addNum` is O(log n), `findMedian` is O(1).
2. **Vary:** the 90th percentile instead of the median. Resize the two heaps to a 90/10 split. A sliding window of the last w values (LeetCode 480). Why do deletions make heaps awkward?
3. **Break the assumptions:** a billion values. Two heaps hold every value. Look up t-digest and reservoir sampling, which give approximate quantiles in fixed memory.
4. **ML lens:** monitoring. A model in production tracks the median and p95 of latency and of each input feature. A drift in the median of a feature is often the first sign that the input data changed.
5. **In the wild:** Python 3.14 added public max-heap functions (`heappush_max`, `heappop_max`, and others) to `heapq`. Before that, only private helpers like `_heappop_max` existed, and most code negated values. Read both versions of `Lib/heapq.py`.

### Shorter ladders: 703, 1046, 215, 621

- **Kth Largest in a Stream.** **Break:** k is larger than the stream so far. **ML lens:** a running leaderboard of the best validation scores across hyperparameter trials.
- **Last Stone Weight.** **Vary:** without negation, using Python 3.14's max-heap functions. **Break:** every stone has the same weight, or there is only one stone. Does your loop return 0 and the single weight correctly?
- **Kth Largest in an Array.** **Vary:** quickselect gives O(n) average and O(n²) worst case. Build the input that triggers the worst case for a first-element pivot. **In the wild:** `np.partition` uses introselect, which falls back to a guaranteed method when quickselect goes badly.
- **Task Scheduler.** **ML lens:** scheduling GPU jobs with cooldowns. **In the wild:** `Lib/asyncio/base_events.py` keeps scheduled callbacks in a heap (`self._scheduled`). Find the `heappush` and the code that rebuilds the heap when too many timers are cancelled.

## ML/DL: neural nets, PyTorch, and backprop

### Learn

- A layer is `y = activation(W x + b)`. A network stacks layers. Without the activation, any stack collapses to one linear layer.
- **Backprop** is the chain rule applied from the loss back to every weight. You built this by hand in Week 3. PyTorch's autograd records the operations and runs the same backward pass.
- The training loop: forward, loss, `zero_grad`, `backward`, `step`. Every line has a failure mode.
- Loss functions: cross-entropy for classes, MSE for regression. Cross-entropy takes raw logits.
- Initialization: weights scaled by `1/sqrt(fan_in)` keep activations from exploding or vanishing as depth grows.
- `model.train()` vs `model.eval()` (dropout, batch norm), and `torch.no_grad()` for inference.
- Batches and `DataLoader`: shuffling, batch size, and the last partial batch.

### Read the code

**1. PyTorch's `nn.Linear`** in `torch/nn/modules/linear.py`. Answer in your log:

- Read `reset_parameters`. It calls `kaiming_uniform_` with `a=math.sqrt(5)`. The comment says this equals uniform `(-1/sqrt(in_features), 1/sqrt(in_features))`. Check that claim numerically for `in_features = 256`.
- What shape is `weight`: `(in, out)` or `(out, in)`? Why does `forward` compute `x @ W.T`?

**2. PyTorch's `CrossEntropyLoss`** in `torch/nn/modules/loss.py`.

- It expects logits, not probabilities. Apply `softmax` first and train anyway. Does it error? What happens to the gradients and the final accuracy?
- What does `ignore_index=-100` do? When would a language model need it?

**3. PyTorch's `SGD`** in `torch/optim/sgd.py`.

- Find where `weight_decay` is added to the gradient, and where the momentum buffer is updated. Write the update as one equation.
- Compare with your Week 3 autograd engine's update step. What does PyTorch do that yours does not?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `nn.Linear` | `bias` | `True` | Adds a learned offset | Without bias, a layer cannot shift its output; fine only if a later norm layer re-centers |
| `nn.CrossEntropyLoss` | `reduction` | `"mean"` | Average, sum, or per-item loss | `"sum"` scales the gradient with batch size, so the right learning rate changes with it |
| `nn.CrossEntropyLoss` | `label_smoothing` | `0.0` | Moves some target mass off the true class | High values make the model unconfident; 0 lets it push logits toward infinity |
| `nn.CrossEntropyLoss` | `ignore_index` | `-100` | Target value that adds no loss | A real class with that label is silently ignored |
| `DataLoader` | `batch_size` | `1` | Items per step | 1 gives noisy, slow training; huge batches can need a higher learning rate and may generalize worse |
| `DataLoader` | `shuffle` | `None` (no shuffle) | Order of items each epoch | Unshuffled sorted data trains on one class at a time |
| `DataLoader` | `drop_last` | `False` | Keep the last partial batch | A batch of 1 breaks batch norm in train mode |
| `optim.SGD` | `lr` | `1e-3` | Step size | Too high diverges to NaN; too low looks like a bug because loss barely moves |
| `optim.SGD` | `momentum` | `0` | Fraction of the last step reused | Near 1 overshoots and oscillates |
| `Optimizer.zero_grad` | `set_to_none` | `True` | Reset gradients to `None` instead of zeros | Code that reads `.grad` after reset gets `None`, not a zero tensor |

### Break it

Remove `optimizer.zero_grad()` from a working training loop. Gradients now accumulate across steps. Plot the loss. Then show that accumulating over 4 steps and dividing the loss by 4 gives the same update as one batch 4 times larger. That is gradient accumulation, used on purpose when a batch does not fit in memory.

## AI agents: RAG, embeddings, and retrieval

### Learn

Read the [RAG chapter](https://mldsa.github.io/docs/ml-spine/chapter-rag/). Then:

- **Why RAG:** the model answers from text you retrieve, so it can cite sources and use data it was not trained on.
- **Chunking:** split documents into pieces with overlap. Chunk size trades context (big chunks) against precision (small chunks).
- **Embeddings:** an embedding model maps text to a vector. Similar meaning gives high cosine similarity (Week 1).
- **Keyword search (BM25):** scores exact term matches, weighted by rarity. Strong on names, codes, and rare words where embeddings are weak.
- **Hybrid search:** run both, then merge the ranked lists (reciprocal rank fusion, from the Merge k Lists ladder).
- **Reranking:** score the top 20 more carefully, keep the top 5.
- **Retrieval evals:** a set of questions with the ID of the chunk that answers each one. **Recall@k** asks "is the right chunk in the top k?" **MRR** asks "how high is it?"
- **Answer evals:** is every claim in the answer supported by a cited chunk? Week 7 automates this with an LLM judge.

### Experiment

Build 30 questions over your corpus, each with its gold chunk ID. Include 5 questions whose answer is not in the corpus. Measure recall@5 and MRR for:

- Chunk sizes of about 128, 256, 512, and 1,024 tokens
- BM25 only, embeddings only, and hybrid

That is a 4×3 table. Which cell wins? Is the winner the same for questions with rare names as for questions phrased in different words from the source?

### Break it

- Ask a question whose answer spans two chunks. Does recall@5 count it as found? Does the answer use both?
- Add 10 near-duplicate copies of one document. They crowd the top 5. Fix it by deduplicating at ingest (row hashes, Week 1) or by limiting results per document.
- Change the embedding model without re-indexing. Queries now live in a different vector space from the chunks. Does anything error? Write a check that stores the model name with the index and refuses a mismatch.
- Ask the 5 unanswerable questions. How many does the system refuse? Set a minimum retrieval score below which it says "not in the documents".

### ML lens

An embedding model is a neural network trained so that matching pairs (a question and its answer) score higher than non-matching ones. That is a contrastive loss, trained with the same loop you read in `torch/optim/sgd.py`. Your reranker this week is a tiny version: an MLP trained on your own eval labels.

### In the wild

- `rank_bm25`: `BM25Okapi` in `rank_bm25.py` takes `k1=1.5` and `b=0.75`. `k1` caps how much repeating a word helps. `b` controls how much long documents are penalized. Set `b=0` and `b=1` and rerun your eval.
- tiktoken's `src/lib.rs`: `_byte_pair_merge_large` uses a `BinaryHeap` to pick the next merge for long inputs. `byte_pair_encode` uses it only when a piece is 100 bytes or longer. Why not always? You will build BPE in [Week 7](week_07.md).

## Software dev: Docker and CI

### Learn

- A Dockerfile: base image, working directory, dependencies, code, command
- Layer caching: copy `requirements.txt` and install before copying the code, so a code change does not reinstall everything
- `.dockerignore`: keep `.env`, `.git`, data, and caches out of the image
- Secrets come in as environment variables at run time, never baked into the image
- A non-root user in the container
- GitHub Actions: a workflow on `push` and `pull_request` that installs, runs `pytest`, and runs the eval
- CI must not call a paid API on every push. Use a small fixture corpus with precomputed embeddings or a fake embedder.

### Apply it

- Write a Dockerfile for this week's build. `docker run` with your API key in an env var must ingest a sample corpus and answer a question.
- Write a CI workflow that runs the tests and the retrieval eval on a fixture of 20 chunks and 10 questions. Fail the job if recall@5 drops below a threshold you choose and justify.

### Verify

- Open a pull request that plants a bug in the ranking (sort ascending instead of descending). CI must go red. Link the failed run in your log.
- Run `docker history` on your image and search the layers for your API key. It must not be there.
- Clone the repo into a fresh folder, build, and run with only the README's steps. Time it.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 36 | Kth Largest in a Stream, Last Stone Weight | Layers, activations; read `nn.Linear` | RAG chapter; chunk your corpus | Dockerfile, layer caching |
| 37 | Kth Largest in an Array | Training loop; read `CrossEntropyLoss` | Embeddings and BM25 over your chunks | `.dockerignore`, secrets, non-root user |
| 38 | K Closest Points | Read `SGD`; Break it: no `zero_grad` | Build the 30-question eval set | GitHub Actions: tests on every PR |
| 39 | Merge k Sorted Lists | `DataLoader`; train an MLP on a toy task | Experiment: chunk size × retriever | Fixture corpus, fake embedder for CI |
| 40 | Task Scheduler, Find Median | Train the reranker MLP on eval labels | Break it: duplicates, model mismatch, refusals | Eval threshold in CI; seeded-bug PR |
| 41 | **Weekly build** | | | |
| 42 | **Defend and publish** | | | |

## Weekly build

### RAG over your problem area, with evals

```bash
rag ingest ./corpus                    # chunk, embed, index
rag ask "how do clinics track no-shows today?"
rag eval evals/questions.jsonl         # recall@k, MRR, refusals
```

The corpus comes from the problem you chose for your top pain in Week 5: public documentation, forum threads, reports, or your own notes. Use interview notes only if you removed names and the person agreed.

**Required behavior**

- **Ingest (AI agents + DSA).**
    - At least 50 documents. Chunk with overlap. Every chunk gets a stable ID.
    - Deduplicate near-identical chunks at ingest.
    - Store the embedding model name with the index, and refuse to query with a different one.
- **Retrieve (DSA + AI agents).**
    - BM25 and embedding search, merged with reciprocal rank fusion.
    - Top-k with a heap or `argpartition`. No full sort of all scores.
- **Rerank (ML/DL).**
    - A small PyTorch MLP on a few features per (question, chunk) pair, such as BM25 score, cosine, and fused rank.
    - Your own training loop, trained on half of the eval questions and measured on the other half.
    - Report whether it beats plain fusion. A reranker that loses is a valid result if you show the numbers.
- **Answer (AI agents).**
    - The top chunks and the question go to the LLM, with instructions to cite chunk IDs.
    - Below a minimum retrieval score, answer "not in the documents" without calling the LLM.
- **Eval (AI agents + Software dev).**
    - At least 30 questions with gold chunk IDs, including at least 5 unanswerable.
    - Reports recall@1, recall@5, MRR, and refusal accuracy, and writes them to a results file.
- **Ship (Software dev).**
    - A Dockerfile and a README a stranger can follow.
    - CI runs tests and the fixture eval on every pull request, and fails below the threshold.

**Stretch**

- Swap exact search for an approximate index (FAISS HNSW or IVF). Plot recall@5 against query time.
- Add a cache of query embeddings. Measure the hit rate on your eval set run twice.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 36 | "PyTorch's `nn.Linear` init uses `a=sqrt(5)`. Here is what that number actually does." Show the numeric check. |
| 37 | "I applied softmax before CrossEntropyLoss. Training still ran." Show the accuracy gap. |
| 38 | "FAISS exact search is LeetCode 973." Walk through `heap_replace_top` in `Heap.h`. |
| 39 | Your 4×3 table: chunk size against BM25, embeddings, and hybrid. Name the winning cell. |
| 40 | "Changing the embedding model without re-indexing raised no error." Show the recall drop and the check you added. |
| 41 | Your RAG build answering a real question from your problem area, with the cited chunk. |
| 42 | What the defense exposed, and what you fixed. |

**This week's write-up:** submit your RAG build to an open **hackathon** whose theme fits your problem area. The entry includes the eval table and the Docker run command. If none is open this week, enter the next one and prepare the submission now.

## Value-creation step

Sketch the MVP that addresses the top pain from Week 5. One page or three screens: what the user does, what they get back, and what it replaces. Show it to two of the people you interviewed. Ask them to walk through it as if it were real, and write down where they hesitate and what they ask for that is not there. Next week you will recruit people to try it.
