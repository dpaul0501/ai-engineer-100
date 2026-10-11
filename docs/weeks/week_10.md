# Week 10 (Days 64–70): Alignment Tables, Generation, and Defending Your Agent

## This week

- **You will learn:**
    - **DSA:** how 2-D dynamic programming aligns two sequences, one cell at a time.
    - **ML/DL:** how a trained model generates text: the KV cache, and the sampling rules that turn logits into tokens.
    - **AI agents:** how prompt injection works against your own app, and how to measure the guardrails you add.
    - **Software dev:** how to find and close the security holes in a small AI product: secrets, dependencies, inputs, and model files.
- **Goal:** by Sunday you can train a small GPT, sample from it with a KV cache that you have proven correct, show that its samples are not copied from the training data, and report the attack success rate of your own agent before and after your defenses.
- **Weekly build:** train and sample a small GPT, with a memorization check and a guardrail evaluation on your MVP. See [Weekly build](#weekly-build).
- **AI rule (Days 61–100):** AI agents, you review. Coding agents write code. You write the spec, review the pull request, run the evals, and reject bad output. You still defend every line on Sunday, including lines an agent wrote.

## DSA: 2-D dynamic programming

**Read first:** [2-D DP patterns](https://mldsa.github.io/docs/part1/chapter03-dp/#4-2d-dp-patterns) and [Beam search: DP for LLM decoding](https://mldsa.github.io/docs/part1/chapter03-dp/#5-beam-search-dp-for-llm-decoding) in *From LeetCode to Learning Systems*.

**Focus.** When the state needs two indices, the table becomes a grid. Most 2-D DPs this week compare two sequences: `dp[i][j]` describes the first `i` items of one and the first `j` of the other. Each cell depends on its left, upper, and upper-left neighbors, so you fill row by row and can usually keep just two rows. Draw the table by hand for a small case before writing code.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 64 | [62. Unique Paths](https://leetcode.com/problems/unique-paths/) | Sum of the cell above and the cell to the left |
| 64 | [64. Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/) | Min instead of sum |
| 65 | [1143. Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/) | Match extends the diagonal; mismatch takes the better neighbor |
| 66 | [72. Edit Distance](https://leetcode.com/problems/edit-distance/) | Insert, delete, or substitute: min of three neighbors |
| 67 | [221. Maximal Square](https://leetcode.com/problems/maximal-square/) | One plus the min of three neighbors |
| 67 | [115. Distinct Subsequences](https://leetcode.com/problems/distinct-subsequences/) | Count alignments, not the best one |
| 68 | [10. Regular Expression Matching](https://leetcode.com/problems/regular-expression-matching/) | `*` means zero copies or one more copy |

### Depth ladder: Edit Distance

1. **Solve.** `dp[i][j]` is the fewest edits to turn `word1[:i]` into `word2[:j]`. The first row and column are 0, 1, 2, and so on. If the characters match, copy the diagonal. Otherwise take 1 plus the min of insert, delete, and substitute. O(mn) time. Then cut space to two rows. Write tests with an empty string on each side first.
2. **Vary.**
    - Only insertions and deletions (LeetCode 583, Delete Operation for Two Strings). How does it relate to LCS?
    - Return the edit script, not just the count. Why does the two-row version lose it?
    - Different costs for each operation. Allow swapping two adjacent characters as one edit (Damerau–Levenshtein).
    - Work on words, not characters. That is word error rate.
3. **Break the assumptions.**
    - **Memory.** Two strings of 10⁵ characters need 10¹⁰ cells for the full table. Two rows fit, but the backtrace is gone. Look up Hirschberg's algorithm, which recovers the script in linear space.
    - **Only small distances matter.** If you need to know only whether the distance is at most k, fill a band of width 2k + 1 around the diagonal. That is O(nk).
    - **Unicode.** `"é"` as one code point vs `e` plus a combining accent gives distance 1 or 2. Normalize first, as in Week 1.
4. **ML lens.**
    - Word error rate for speech recognition and character error rate for OCR are edit distances, divided by the reference length.
    - CTC loss for speech models is a 2-D DP over time steps and label positions. It sums over alignments, like Distinct Subsequences, instead of taking the best one.
    - Fuzzy matching of LLM outputs against expected answers in your eval set (Week 2) often uses normalized edit distance.
5. **In the wild.**
    - `torchaudio.functional.edit_distance` in `src/torchaudio/functional/functional.py`. It is the two-row version in pure Python. Benchmark it against yours.
    - RapidFuzz: `rapidfuzz.distance.Levenshtein.distance`. Benchmark it on the same inputs. Find which algorithm its documentation says it uses, and explain the speedup in one paragraph.

### Depth ladder: Longest Common Subsequence

1. **Solve.** If `a[i-1] == b[j-1]`, `dp[i][j] = dp[i-1][j-1] + 1`. Otherwise take the max of the cell above and the cell to the left. Then backtrack from the corner to recover the subsequence.
2. **Vary.**
    - Longest common **substring**, where characters must be contiguous. A mismatch resets the cell to 0, and the answer is the max anywhere in the table, not the corner.
    - Shortest common supersequence (LeetCode 1092). Build it from the LCS table.
    - Three strings. What is the cost in time and memory?
3. **Break the assumptions.**
    - Ties in the backtrack. Your function may return a different valid LCS from the expected one. Test the length, and test that the result is a subsequence of both, rather than comparing strings.
    - Two files of 10⁵ lines. The table does not fit. Real diff tools use Myers' O(ND) algorithm, where D is the number of differences.
4. **ML lens.**
    - ROUGE-L, a summarization metric, is the LCS of the candidate and reference, turned into precision, recall, and F-measure.
    - **Memorization.** Longest common substring between a generated sample and the training data shows whether the model is copying. This week's build uses it.
5. **In the wild.**
    - `_lcs_table` and `_score_lcs` in `rouge/rouge_scorer.py` in Google Research's `rouge` package. How are tokens normalized before the LCS?
    - Python's `difflib.SequenceMatcher` in `Lib/difflib.py` is **not** LCS. It finds the longest matching block, then recurses on each side. Find `autojunk` in the constructor. On what inputs does it change the result?
    - git's default diff: `xdiff/xdiffi.c`, an implementation of Myers' algorithm. Compare it with patience diff from Week 9.

### Depth ladder: Regular Expression Matching

1. **Solve.** `dp[i][j]` is true if `s[:i]` matches `p[:j]`. For `x*`, either use zero copies (`dp[i][j-2]`) or, if `x` matches `s[i-1]`, one more copy (`dp[i-1][j]`). Initialize the first row carefully for patterns like `a*b*`.
2. **Vary.** Wildcard matching with `?` and `*` (LeetCode 44). Which transitions change?
3. **Break the assumptions.**
    - A pattern that starts with `*`. The problem forbids it. What does your code do?
    - Compare your DP with Python's backtracking `re` from Week 8 on `re.fullmatch("a*a*a*a*a*b", "a" * n)` for n = 20, 40, 80, 160. Each `a*` is one more choice point, so backtracking time grows like a high power of n. The DP stays O(mn).
4. **ML lens.** Constrained decoding for regular expressions does not backtrack. It compiles the pattern into a finite-state machine, then at each step masks every token that has no valid transition from the current state (Willard and Louf, 2023, [arXiv:2307.09702](https://arxiv.org/abs/2307.09702)). That is the `-inf` mask from Week 8, driven by a state machine.
5. **In the wild.** Go's `regexp` package and Google's RE2 guarantee time linear in the input length by never backtracking. They give up backreferences to get that guarantee. Read the package documentation for `regexp` and find the sentence that states it.

### Shorter ladders: Unique Paths, Minimum Path Sum, Maximal Square, Distinct Subsequences

- **Unique Paths.** Vary: obstacles (LeetCode 63, Unique Paths II). Break: the closed form is the binomial coefficient C(m + n − 2, m − 1). Compute it with `math.comb` and with floats, and find where floats go wrong.
- **Minimum Path Sum.** ML lens: dynamic time warping, which aligns two time series of different speeds, uses the same recurrence with a min over three neighbors. Break: negative cells. Does the DP still hold if moves are only right and down?
- **Maximal Square.** Break: the input is a grid of strings `"0"` and `"1"`, not integers. Does your code compare against the right type? Vary: largest rectangle of 1s (LeetCode 85), which needs a stack per row.
- **Distinct Subsequences.** Counts grow fast. Python is fine, but the same code with a NumPy int64 table overflows silently. Show the input that does it. ML lens: summing over all alignments is what CTC loss does.

## ML/DL: inference, the KV cache, and sampling

### Learn

- Training vs inference: one parallel pass over known tokens vs one new token at a time ([inference chapter](https://dpaul0501.github.io/llm-agent-reads/part_06_inference_and_tricks/))
- Prefill (process the whole prompt) and decode (one token per step)
- The KV cache: store each layer's keys and values so decoding computes attention for the new token only
- KV cache memory: 2 × layers × heads × head size × sequence length × batch × bytes per value
- Greedy decoding, temperature, top-k, top-p (nucleus), and repetition penalty
- Beam search as pruned search over sequences ([DP chapter](https://mldsa.github.io/docs/part1/chapter03-dp/#5-beam-search-dp-for-llm-decoding))
- What FlashAttention and speculative decoding change, and what they leave the same

### Read the code

**1. This repo's [`inference.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/inference.py).** Answer in your log:

- The buffers are allocated as float64. Pass float32 keys. What dtype comes back from `get`, and what does that do to memory?
- `get` returns slices of the internal buffers. Modify the array it returns. What happens to the cache?
- `update` checks the shape of `k` but not of `v`. Write the call that stores mismatched values without an error.
- `test_kv_cache_updates_and_shape` checks shapes after one update. Change `update` to always write at index 0. Does the test still pass? Write a test that fails.
- The cache does not know positions. With RoPE from Week 8, what position must the new token's query and key be rotated by?
- Now compare the toy `KVCache` in the inference chapter. When it is full, it drops the oldest entry. The repo's version raises. Which is safer, and what does dropping entries change about what the model sees?

**2. nanoGPT's `generate`** in [`model.py`](https://github.com/karpathy/nanoGPT/blob/master/model.py) and [`sample.py`](https://github.com/karpathy/nanoGPT/blob/master/sample.py).

- `generate` has no KV cache. It reruns the model on the full context for every new token. What is the total cost for N new tokens?
- It crops the context to `block_size`. What happens to a long prompt?
- `logits[:, -1, :] / temperature`. What happens at `temperature = 0`?
- Top-k keeps every logit not below the k-th largest: `logits[logits < v[:, [-1]]] = -float('Inf')`. With ties at the k-th place, how many tokens survive?

**3. Hugging Face `transformers`: `generation/logits_process.py`.**

- `TemperatureLogitsWarper.__init__` raises on `0.0` and tells you to use greedy decoding instead. Compare that with nanoGPT.
- `TopPLogitsWarper.__call__` sorts ascending, takes a cumulative sum of probabilities, and removes tokens at or below `1 − top_p`. Then `min_tokens_to_keep` protects the last ones. Trace it on `[0.5, 0.3, 0.2]` with `top_p = 0.6`.
- `RepetitionPenaltyLogitsProcessor` divides positive logits by the penalty and multiplies negative ones. Why not just subtract a constant?

**4. Hugging Face `cache_utils.py`.** `DynamicLayer.update` grows the cache with `torch.cat` on every step. `StaticLayer` preallocates. Which one does the llm-agent-reads version resemble, and what does each cost over 1,000 steps?

**5. Back to Week 8.** PyTorch's `scaled_dot_product_attention` with `is_causal=True` builds a top-left aligned mask. During cached decoding there is one query and many keys. Explain why you must pass `is_causal=False` (or an explicit mask) for decode steps.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `KVCache` (llm-agent-reads) | `max_len` | required | Preallocated sequence length | Too small: generation fails mid-answer. Too large: memory is reserved for tokens that never come |
| `KVCache` (llm-agent-reads) | `batch_size` | `1` | Sequences cached together | Every sequence must be the same length. Mixed lengths need padding and masks |
| nanoGPT `generate` | `temperature` | `1.0` | Divides the logits before softmax | Near 0: greedy and repetitive, and exactly 0 divides by zero. High: near-uniform noise |
| nanoGPT `generate` | `top_k` | `None` | Keep only the k highest logits | `1` is greedy. `None` keeps the long tail of unlikely tokens. Ties keep more than k |
| nanoGPT `sample.py` | `temperature`, `top_k` | `0.8`, `200` | The script's sampling settings | These differ from `generate`'s defaults. Report which ones produced your samples |
| nanoGPT `sample.py` | `seed` | `1337` | The random seed | A fixed seed makes samples look stable. Report results over several seeds |
| `TopPLogitsWarper` (HF) | `top_p` | required | Smallest set of tokens whose probability reaches `top_p` | `1.0` keeps everything. Very small keeps one token, which is greedy |
| `TopPLogitsWarper` (HF) | `min_tokens_to_keep` | `1` | Floor on surviving tokens | Without a floor, a strict threshold could remove every token |
| `TopKLogitsWarper` (HF) | `filter_value` | `-float("Inf")` | What removed logits are set to | A finite value like `-1e9` leaves removed tokens with a tiny nonzero probability |
| `RepetitionPenaltyLogitsProcessor` (HF) | `penalty` | required (`1.0` means none) | How much to discourage tokens already seen | High values ban common words like "the" and break grammar |

### Break it

- **Cached vs uncached.** At every decode step, the logits from your cached path must match a full recompute to within float tolerance. Plant the bug where the new token's RoPE position is 0. The test must fail.
- **Measure the cache.** Time tokens per second with and without the cache at 64, 256, and 1,024 generated tokens. Compute the cache size from the formula, then measure it. Do they agree?
- **Ties in top-k.** Build logits with five equal values at the k-th place. Count survivors in nanoGPT's top-k and in yours.
- **Same seed, different batch.** Generate with the same seed at batch size 1 and 4. Are the first sequence's tokens identical? Explain why or why not.

## AI agents: guardrails and prompt injection

All testing this week is against **your own app**, in your own environment, with stub tools that cannot cause harm.

### Learn

- **Direct injection:** the user types instructions that override yours.
- **Indirect injection:** instructions hidden in content the agent reads: a retrieved document, a web page, a tool result, or an MCP tool description.
- Why it is hard: the model sees one stream of tokens. Nothing in the input marks which tokens are instructions and which are data.
- The [OWASP Top 10 for LLM Applications](https://genai.owasp.org/llm-top-10/). Start with LLM01 Prompt Injection, LLM06 Excessive Agency, and LLM07 System Prompt Leakage.
- Defenses that reduce harm even when injection succeeds: least privilege for tools, human confirmation before side effects, no automatic rendering of links or images from model output, and output validation against a schema.
- Defenses that try to detect injection: input and output classifiers, and allowlists. They have false positives and false negatives, like any classifier.
- The dangerous combination: an agent with access to private data, exposure to untrusted content, and a way to send data out. Remove one of the three wherever you can.

### Experiment

Build a test suite for your MVP's agent:

- 30 injection cases: 10 direct, 10 planted in documents in your own retrieval corpus, and 10 returned by a test tool on your own MCP server.
- 30 benign requests that look similar, such as a document that discusses prompt injection without containing one.
- A harmless canary for success: the attack "wins" if the agent calls a stub tool such as `send_message`, which only logs, or if a canary string from your system prompt appears in the output.

Measure the attack success rate and the false-positive rate with no defenses. Then add one defense at a time and measure both again. Report a table: defense, attack success rate, false-positive rate, added latency.

### Break it

- Wrap retrieved documents in delimiters and tell the model to ignore instructions inside them. Rerun the suite. How much did delimiters alone help?
- Rephrase your three most successful attacks: translate them, split them across two documents, or bury them in a long, relevant document. Does your classifier still catch them?
- Turn on the human-confirmation step for `send_message`. The attack success rate for that tool should drop to zero. What did the extra step cost the real users?

### ML lens

Attention, from Week 8, mixes every token in the context. An instruction inside a retrieved document gets the same machinery as your system prompt. A detection guardrail is a classifier with a precision-recall tradeoff, like the ones you evaluated in Week 4. Pick its threshold from your measured false-positive rate, not from a default.

### In the wild

NVIDIA's open-source scanner `garak` has a module of prompt-injection probes buried in documents: `garak/probes/latentinjection.py`. Read `_build_prompts_triggers`: each test case combines a task, a context document with an injection marker, an injected instruction, and a trigger string that the detector looks for in the output. Adapt three of its patterns to your own suite.

## Software dev: security

### Learn

- **Secrets:** keep API keys in environment variables or a secrets manager, never in the repo. A key that was ever committed is leaked, even after you delete it. Rotate it.
- **Dependencies:** pin versions, and audit them with `pip-audit`.
- **Inputs:** validate at every boundary: HTTP requests, tool arguments, file uploads, and URLs your agent fetches.
- **Model files:** `.pt` files are pickles, and loading a pickle can run code. Since PyTorch 2.6, `torch.load` defaults to `weights_only=True`. Save weights with `safetensors` where you can.
- **Access:** authentication for your MVP, per-user rate limits, and API keys with the smallest scope that works.
- **Threat model:** list assets, entry points, and trust boundaries before you list fixes.

### Apply it

Write a one-page threat model for your MVP. For each entry point, name the worst thing an attacker could do and the control that stops it. Then fix the top three.

### Verify

- Run a secrets scanner such as `gitleaks` over your **full git history**, not just the working tree. Record and rotate anything it finds.
- Run `pip-audit`. Fix or document every finding.
- Create a pickle file whose payload only writes a harmless marker file. Load it with `torch.load(..., weights_only=False)` and then with the default. Record which one wrote the marker, and make sure your code never uses the first form on files it did not create.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 64 | Unique Paths, Minimum Path Sum | Prefill vs decode; read `inference.py` | Direct and indirect injection; OWASP LLM01 | Secrets; scan full history |
| 65 | Longest Common Subsequence | KV cache with RoPE positions; cached vs uncached test | Write the 60-case suite and canaries | Dependency audit, pinning |
| 66 | Edit Distance | Sampling; read nanoGPT `generate` and HF `logits_process.py` | Baseline attack success rate | Input validation at every boundary |
| 67 | Maximal Square, Distinct Subsequences | Measure cache speed and memory | Add defenses one at a time | Model files: pickle, `weights_only`, safetensors |
| 68 | Regular Expression Matching | Train the small GPT; sweep sampling settings | Rephrased attacks; tune the classifier threshold | Threat model; fix the top three |
| 69 | **Weekly build** | | | |
| 70 | **Defend and publish** | | | |

## Weekly build

### Train and sample a small GPT

```bash
python -m gpt.train --config configs/small.yaml
python -m gpt.sample --prompt "..." --temperature 0.8 --top-k 50 --top-p 0.95 --seed 1
python -m gpt.bench --lengths 64 256 1024            # cached vs uncached
python -m gpt.memorize --samples out/samples.jsonl --train data/train.txt
python -m guard.eval --suite evals/injection/        # attack success rate, before and after
```

**Required behavior**

- **Model and sampling (ML/DL).**
    - Train a small GPT, a few layers wide enough to produce readable text, on a corpus from your product's domain. Report train and validation loss curves.
    - A sampler with greedy, temperature, top-k, and top-p. Invalid settings raise clear errors. Temperature 0 means greedy, explicitly.
    - Generation uses a KV cache with correct RoPE positions. Cached logits match a full recompute at every step.
    - A benchmark of tokens per second with and without the cache.
- **Memorization check (DSA).**
    - For each sample, find the longest common substring with the training text.
    - Shortlist candidate regions with n-gram hashing (Week 1), then run the 2-D DP only on those regions.
    - Flag any sample with an overlap longer than a threshold you justify. Report the flag rate for each sampling setting.
- **Guardrails (AI agents).**
    - The 60-case suite against your MVP's agent, with the before-and-after table.
    - At least one defense that limits harm even when detection fails, such as confirmation before side effects.
- **Security (Software dev).**
    - The threat model, a clean secrets scan of full history, and `pip-audit` results.
    - Checkpoints saved as safetensors, or loaded only with `weights_only=True`.

**Stretch**

- Add beam search. Compare its samples with top-p on repetition and on memorization flags.
- Store the KV cache in float16 and measure the largest logit error it introduces.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 64 | "A KV cache test that passes when every token is written to slot 0." Show the shape test and the one that catches it. |
| 65 | "ROUGE-L is Longest Common Subsequence." Link `rouge_scorer.py`. |
| 66 | "nanoGPT's top-k keeps more than k tokens when there are ties." Show the case. |
| 67 | "I measured my KV cache: formula vs reality." The table. |
| 68 | "I attacked my own agent 30 ways. Here is what each defense actually stopped." |
| 69 | Samples from your small GPT, with the memorization flag rate per setting. |
| 70 | What the defense exposed, and what you fixed. |

**This week's write-up:** a Kaggle notebook that trains your small GPT on a public dataset and sweeps temperature and top-p, plotting validation loss, sample diversity, and the memorization flag rate together.

## Value-creation step

!!! note "Your paths"
    This step is written for the Product path. On other paths, use this time for your next milestone in [Choose your paths](../paths.md).

Talk to each active user from Week 9. Ask one direct question: what would they pay for this, or what would they trade for it (time, data, an introduction, a testimonial)? Write down the exact answers, including "nothing". Pick the 2 users with the strongest answers as candidates for a paid pilot, and write one sentence each on what the pilot would include.
