# Week 9 (Days 57–63): Recurrences, the Transformer Block, and Tools Over a Protocol

## This week

- **You will learn:**
    - **DSA:** how 1-D dynamic programming turns an exponential recursion into a single pass over a table.
    - **ML/DL:** how a decoder block is assembled, and how next-token training actually updates it.
    - **AI agents:** how the Model Context Protocol (MCP) lets any agent discover and call your tools.
    - **Software dev:** how logs, metrics, and traces let you answer questions about a running system without guessing.
- **Goal:** by Sunday you can train a small decoder-only transformer, prove its forward pass matches your NumPy block, reproduce one published claim about it, and show from your own logs that 10 people used your MVP.
- **Weekly build:** the **Month 2 milestone**: an arXiv-style technical report on one reproduced claim, plus your MVP exposed as an MCP server and instrumented end to end. See [Weekly build](#weekly-build) and [Month 2 milestone](#month-2-milestone).
- **AI rule: this week the rule changes.**
    - **Days 57–60: AI pairs, you prove.** AI may write code. Tests come first, and you explain every line it wrote.
    - **Days 61–63: AI agents, you review.** From Day 61, coding agents write code. You write the spec, review the pull request, run the evals, and reject bad output.
    - Write the training loop and the NumPy-to-PyTorch weight port before Day 61. Hand the MCP server and the logging plumbing to an agent on Day 61, as your first spec-and-review cycle.

## DSA: 1-D dynamic programming

**Read first:** [1-D DP patterns](https://mldsa.github.io/docs/part1/chapter03-dp/#3-1d-dp-patterns) and [Viterbi: DP for sequence labeling](https://mldsa.github.io/docs/part1/chapter03-dp/#6-viterbi-dp-for-sequence-labeling) in *From LeetCode to Learning Systems*.

**Focus.** Last week's backtracking explored every branch. When different branches reach the same subproblem, you can solve it once and store the answer. That is dynamic programming. A 1-D DP has one index of state, usually "the first i items" or "amount i". Each problem this week needs four decisions: what `dp[i]` means, the recurrence, the base case, and the order of filling. Most bugs are in the first one.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 57 | [70. Climbing Stairs](https://leetcode.com/problems/climbing-stairs/) | Fibonacci recurrence, O(1) space |
| 57 | [198. House Robber](https://leetcode.com/problems/house-robber/) | Take this one or skip it |
| 58 | [213. House Robber II](https://leetcode.com/problems/house-robber-ii/) | Break the circle into two lines |
| 58 | [322. Coin Change](https://leetcode.com/problems/coin-change/) | Minimum over the last coin used |
| 59 | [139. Word Break](https://leetcode.com/problems/word-break/) | `dp[i]`: can the prefix of length i be segmented? |
| 60 | [91. Decode Ways](https://leetcode.com/problems/decode-ways/) | Count segmentations; zeros are the trap |
| 61 | [300. Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/) | O(n²) DP, then O(n log n) with binary search |

### Depth ladder: Coin Change

1. **Solve.** `dp[a]` is the fewest coins that make amount `a`, with `dp[0] = 0` and everything else infinity. For each amount, take the minimum over coins of `dp[a − c] + 1`. Return −1 if `dp[amount]` is still infinity. O(amount × coins) time, O(amount) space. Write tests for amount 0 and an unreachable amount first.
2. **Vary.**
    - Count the ways to make the amount, where order does not matter (LeetCode 518, Coin Change II). Swap the loop order and you count ordered sequences instead. Show both numbers for coins `[1, 2]`, amount 3.
    - Return the coins used, not just the count. What extra array do you need?
    - Each coin may be used at most once. The 1-D table must now be filled from high to low. Why?
3. **Break the assumptions.**
    - **Greedy.** Coins `[1, 3, 4]`, amount 6. Greedy takes 4 + 1 + 1. The answer is 3 + 3. Which coin systems is greedy safe for?
    - **Top-down recursion.** Write it with `@functools.lru_cache` and run amount 10,000 with coin 1. You hit Python's recursion limit. Then notice that `lru_cache`'s default `maxsize` is 128. What does that do to a memoized DP?
    - **Huge amounts.** Amount 10⁹ does not fit in a table. What would you do instead?
4. **ML lens.**
    - `dp[a] = min(dp[a − c] + 1)` is a Bellman equation. Value iteration in reinforcement learning fills the same kind of table with expected rewards ([DP chapter](https://mldsa.github.io/docs/part1/chapter03-dp/#7-rl-value-iteration-dp-as-planning)).
    - Coin Change is also a shortest path from 0 to `amount` in a graph of amounts. Solve it with the BFS from Week 7 and compare.
5. **In the wild.** `functools.lru_cache` in CPython's `Lib/functools.py`. Read `_make_key`. Why does a list argument raise `TypeError`? Why does the cache treat `f(1)` and `f(1.0)` as the same call unless `typed=True`? Then read `cache`, added in Python 3.9, and say when you would use it instead.

### Depth ladder: Word Break

1. **Solve.** `dp[i]` is true if `s[:i]` can be split into dictionary words. `dp[0]` is true. For each `i`, check every `j < i` with `dp[j]` true and `s[j:i]` in the word set. Bound the inner loop by the longest word length.
2. **Vary.**
    - Return every segmentation (LeetCode 140, Word Break II). The output can be exponential. This is backtracking with memoization: Week 8 and Week 9 together.
    - Give each word a score, and find the segmentation with the highest total. Replace `or` with `max` and `True` with a score.
    - Count the segmentations instead. Replace `or` with `+`.
3. **Break the assumptions.**
    - `s = "a" * 30 + "b"`, words `["a", "aa", "aaa"]`. A plain recursion without memoization makes tens of millions of calls. Count the calls with and without memoization.
    - Case and Unicode: does `"Apple"` match `"apple"`? Normalize both sides the way you did in Week 1.
    - A dictionary of 10⁶ words. The word set is fine, but `s[j:i]` creates a new string for every check. Use the trie from Week 5 to walk forward from each `j` instead.
4. **ML lens.** This is tokenization.
    - A Unigram tokenizer gives every vocabulary piece a log-probability, then picks the segmentation with the highest total. That is Word Break with scores, solved by the Viterbi algorithm.
    - Counting segmentations with `+` is the forward algorithm. In log space, `+` becomes `logsumexp`, which subtracts the max first, like `softmax` in Week 1.
    - BPE, which you built in Week 7, applies merges greedily and does not search for the best segmentation. Compare the two on your corpus in this week's build.
5. **In the wild.** SentencePiece's `Lattice::Viterbi` in `src/unigram_model.cc`. Find the forward pass that fills the best score for each position, and the backtrace that recovers the pieces. Hugging Face `tokenizers` has the same idea in its Unigram model's `lattice.rs`.

### Depth ladder: Longest Increasing Subsequence

1. **Solve.** First the O(n²) DP: `dp[i]` is the length of the longest increasing subsequence ending at `i`. Then the O(n log n) version: keep `tails[k]`, the smallest tail of any increasing subsequence of length k + 1, and update it with `bisect_left`.
2. **Vary.**
    - Non-decreasing instead of strictly increasing. Which `bisect` function changes?
    - Return the subsequence itself, not its length. `tails` is not the answer. Prove it with `[3, 4, 1]`.
    - Envelopes that nest in two dimensions (LeetCode 354, Russian Doll Envelopes). Why must equal widths be sorted by height descending?
3. **Break the assumptions.** Duplicates, an empty array, and a strictly decreasing array of 10⁵ elements. Time the O(n²) version on it.
4. **ML lens.** Diffing two model outputs, two configs, or two versions of a prompt needs a good alignment of lines. Patience diff aligns on unique lines using LIS. Monotonic constraints also appear in isotonic regression for calibrating classifier scores.
5. **In the wild.** git's patience diff: `find_longest_common_sequence` and `binary_search` in `xdiff/xpatience.c`. It is your `tails` array with pointers back for reconstruction. Run `git diff --patience` on a file where the default diff looks confusing.

### Shorter ladders: Climbing Stairs, House Robber, House Robber II, Decode Ways

- **Climbing Stairs.** Vary: steps of size 1, 2, or 3, or a set of allowed sizes. Break: n = 10⁶ with the naive recursion. ML lens: a linear recurrence can be computed by matrix powers in O(log n). The same idea lets linear recurrent models be computed in parallel.
- **House Robber.** Vary: houses on a binary tree (LeetCode 337, House Robber III). Break: negative values. Should the robber ever take one?
- **House Robber II.** The circle couples the first and last house. Solve two straight-line problems and take the max. Break: a single house, where both sub-problems are empty.
- **Decode Ways.** Break: `"06"`, `"100"`, `"0"`. Write those tests before the code. Vary: add `*` as a wildcard (LeetCode 639, Decode Ways II), where the answer is taken modulo 10⁹ + 7. Python integers do not overflow. What would happen in C or in a NumPy int64 array?

## ML/DL: the transformer block and training

### Learn

- The pre-norm decoder block: LayerNorm, attention, residual add, LayerNorm, MLP, residual add ([transformer block chapter](https://dpaul0501.github.io/llm-agent-reads/part_04_transformer_block/))
- The residual stream, LayerNorm over the feature axis, the 4× MLP expansion, and GELU
- Counting parameters by hand, then matching the count in code
- The final LayerNorm, the vocabulary projection, and weight tying
- Next-token prediction: shifted targets, teacher forcing, and cross-entropy ([training chapter](https://dpaul0501.github.io/llm-agent-reads/part_05_training_and_optimization/))
- AdamW, learning-rate warmup and cosine decay, gradient clipping, and gradient accumulation
- The [LLM chapter](https://mldsa.github.io/docs/ml-spine/chapter-llm/) in *From LeetCode to Learning Systems* for the larger picture

### Read the code

**1. This repo's [`decoder_block.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/decoder_block.py) and its test in [`tests/test_foundations.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/tests/test_foundations.py).** Answer in your log:

- Is this block pre-norm or post-norm? Point to the line that decides it.
- `DecoderBlock(seed=...)` passes the seed only to the MLP. `MultiHeadAttention` always uses `np.random.default_rng(0)`. Build two blocks with different seeds and compare their `W_q`. What would that mean for a 12-block stack?
- `LayerNorm.__call__` adds `eps` inside the square root. What happens to a token whose features are all equal if you move `eps` outside?
- `test_decoder_block_preserves_residual_shape_and_causality` changes the last token and checks earlier outputs. That is a good test. Plant each bug below and record whether it catches it:
    - LayerNorm over the time axis instead of the feature axis
    - post-norm instead of pre-norm
    - one residual connection removed

**2. nanoGPT's [`model.py`](https://github.com/karpathy/nanoGPT/blob/master/model.py).**

- `Block.forward` is two lines. Match each to a line of `DecoderBlock.__call__`.
- `_init_weights` uses standard deviation 0.02. Then the constructor rescales every `c_proj.weight` by `1/√(2 · n_layer)`. Why do residual projections get a smaller init?
- Find the weight-tying line, `self.transformer.wte.weight = self.lm_head.weight`. How many parameters does it save at GPT-2 small sizes (vocabulary 50,257 padded to 50,304, width 768)?
- `configure_optimizers` applies weight decay only to parameters with 2 or more dimensions. Which parameters escape decay, and why should they?
- In `forward`, with no targets, the model projects only the last position to the vocabulary. Why is that safe at inference and wrong during training?

**3. nanoGPT's [`train.py`](https://github.com/karpathy/nanoGPT/blob/master/train.py).**

- `get_lr`: linear warmup, cosine decay, then a floor. Plot it for the default settings.
- In the accumulation loop, the loss is divided by `gradient_accumulation_steps` before `backward`. What goes wrong if you remove that division?
- `clip_grad_norm_` is called after `scaler.unscale_`. Why must the order be that way when float16 uses a `GradScaler`?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `LayerNorm` (llm-agent-reads) | `eps` | `1e-5` | Floor under the variance | Too small: a constant token divides by nearly zero. Too large: features are no longer normalized |
| `DecoderBlock` (llm-agent-reads) | `d_ff` | required | MLP hidden width, usually 4 × `d_model` | Too small: a bottleneck that limits capacity. Too large: most parameters and compute sit in the MLP |
| `DecoderBlock` (llm-agent-reads) | `seed` | `0` | Random init of the MLP only | Different seeds still give identical attention weights in every block |
| `torch.optim.AdamW` | `lr` | `1e-3` | Step size | Too high: loss spikes or goes to NaN. Too low: the run looks stable and never learns |
| `torch.optim.AdamW` | `betas` | `(0.9, 0.999)` | Decay of the gradient mean and variance estimates | nanoGPT uses `beta2 = 0.95`. A high `beta2` reacts slowly when gradient scale changes, which can make large-model training unstable |
| `torch.optim.AdamW` | `weight_decay` | `1e-2` | Pull of weights toward zero | Too high: underfits. Applied to LayerNorm gains and biases, it shrinks parameters that should not shrink |
| `torch.optim.AdamW` | `eps` | `1e-8` | Floor in the update's denominator | Too large: AdamW behaves like SGD with momentum. In low precision, a tiny eps can underflow |
| nanoGPT `train.py` | `warmup_iters` | `2000` | Steps of linear learning-rate warmup | Zero warmup with a high learning rate can diverge in the first steps, especially in post-norm blocks |
| `torch.nn.utils.clip_grad_norm_` | `max_norm` | required (nanoGPT: `1.0`) | Largest allowed total gradient norm | Too low: every step is clipped and the learning rate is effectively lower. Too high: does nothing during spikes |
| `torch.nn.utils.clip_grad_norm_` | `error_if_nonfinite` | `False` | Raise on NaN or inf gradients | With the default, a NaN gradient can pass through silently |
| `F.cross_entropy` | `ignore_index` | `-100` | Target value that contributes no loss | nanoGPT passes `-1`. A padding id that does not match trains the model to predict padding |
| `F.cross_entropy` | `label_smoothing` | `0.0` | Mass moved from the true token to all tokens | High values cap confidence. Loss can no longer reach zero, so overfitting tests need it off |

### Break it

- **Loss at init.** Before any training, the loss on random weights should be close to `ln(vocab_size)`. Measure yours. If it is far off, find out why before training.
- **Forget the shift.** Use `targets = inputs` instead of inputs shifted by one. The loss falls almost to zero within a few hundred steps. Explain what the model learned, and why the causal mask makes it easy.
- **Overfit one batch.** A correct model must drive the loss near zero on a single batch. If it cannot, you have a bug. This is the first test to run on any new model.
- **Remove the residuals** in an 8-block stack. Log the gradient norm of the first block's weights at step 0 with and without residuals.

## AI agents: MCP and tool ecosystems

### Learn

- Why a protocol: one tool server works with any MCP-capable host, instead of one integration per agent framework
- Host, client, and server. The server exposes three kinds of things: **tools** (actions the model calls), **resources** (data the application reads), and **prompts** (reusable templates)
- Messages are JSON-RPC 2.0. The two standard transports are **stdio** (the host launches the server as a subprocess) and **Streamable HTTP**
- A tool definition: `name`, `description`, and `inputSchema` (JSON Schema), with optional `outputSchema` and annotations
- Two kinds of errors: protocol errors (unknown tool, malformed request) and tool execution errors, returned in the result with `isError: true` so the model can correct itself
- Read the [specification](https://modelcontextprotocol.io/specification/latest). Check which revision is current. The 2026-07-28 revision carries the protocol version and capabilities on every request; earlier revisions opened a session with an `initialize` handshake
- The official [Python SDK](https://github.com/modelcontextprotocol/python-sdk). In version 2 the high-level server class is `MCPServer` (it was `FastMCP` in version 1). Tools are plain typed functions with a decorator

### Experiment

Expose your MVP's core feature as an MCP server with three tools. Connect it to an agent through an MCP client. Write 20 user requests with the correct tool for each, then measure tool-selection accuracy three ways:

- with clear descriptions
- with vague, one-word descriptions
- with clear descriptions plus 20 unrelated distractor tools

Also log input tokens per request in each setup. Tool definitions are part of the prompt. How much context do 23 tools cost?

### Break it

- Add a `print("debug")` inside a tool in a stdio server. The spec says the server must not write anything to stdout that is not a valid MCP message. What happens with your SDK version? Then log to stderr instead.
- Raise an exception inside a tool, then return the same failure as a tool result with `isError: true`. Does the agent recover differently?
- Return 1 MB of text from a tool. What does the agent do with it, and what does it cost?
- Connect two servers that both expose a tool named `search`. How does your client tell them apart? The spec suggests prefixing names with a server identifier.

### ML lens

Tool names, descriptions, and schemas become tokens in the model's context. Choosing a tool is next-token prediction conditioned on those tokens, so description quality matters the way prompt quality does. Arguments that must match `inputSchema` are the constrained-decoding problem from Week 8's Generate Parentheses. Because descriptions are read as prompt text, a server you do not control can put instructions in them. Week 10 tests that against your own app.

### In the wild

The Python SDK, `src/mcp/server/stdio.py`. Read `_claim_fd`. Version 2 moves the protocol stream to a private file descriptor and points file descriptor 1 at stderr. What problem does that solve, and what did version 1 do instead? Then read `src/mcp/server/mcpserver/utilities/func_metadata.py`: how does a Python function signature become an `inputSchema`, and where are arguments validated?

## Software dev: logging and observability

### Learn

- Python `logging`: loggers, handlers, levels, and why libraries should not configure the root logger
- Structured logs: one JSON object per event, with a request ID that ties events together
- Metrics: counters and histograms; p50 and p95 latency, not averages
- Traces and spans: where time goes inside one request. OpenTelemetry is the open standard, and it defines GenAI conventions (`gen_ai.*` attributes) for model calls
- What to log for an LLM feature: request ID, a hashed user ID, input and output tokens, latency, stop reason, tool calls, and outcome
- What never to log: API keys, raw personal data, and full prompts that contain user secrets. Redact before writing

### Apply it

Instrument two things this week:

- **The training run.** Log step, loss, learning rate, gradient norm, and tokens per second as JSON lines. Every figure in your report is drawn from these logs by a script.
- **The MVP.** Log every request with the fields above. This is how you count your 10 users honestly.

### Verify

Answer five questions using only your logs, with no debugger and no guessing:

- How many distinct users completed the core task this week?
- What was the p95 latency, and which step dominated it?
- Which request failed most recently, and why?
- How many tokens did the median request use?
- Did gradient norm spike before any loss spike in training?

If you cannot answer one, add the missing field and say so in your log.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 57 | Climbing Stairs, House Robber | Block pipeline; read `decoder_block.py` | MCP architecture: host, client, server | `logging` basics, levels, handlers |
| 58 | House Robber II, Coin Change | Residuals, LayerNorm, parameter count; read nanoGPT `Block` | First MCP server over stdio | Structured JSON logs, request IDs |
| 59 | Word Break | Next-token loss, shifted targets, loss at init | Client connection; tool errors with `isError` | Metrics: counters, histograms, p95 |
| 60 | Decode Ways | AdamW, warmup, clipping; read `train.py` | Tool-selection experiment | **Month 2 milestone check:** count 10 users from logs; report draft with results |
| 61 | Longest Increasing Subsequence | Overfit one batch; run the reproduction | **AI rule switch:** an agent writes the MCP server from your spec; you review the PR | Traces; redaction review of the agent's logging |
| 62 | **Weekly build** | | | |
| 63 | **Defend and publish** | | | |

## Weekly build

### Month 2 technical report: one claim about the transformer block, reproduced

```bash
python -m tiny.train --config configs/pre_ln.yaml --seed 0     # JSON-lines log per step
python -m tiny.port_check                                      # PyTorch block == NumPy block
python -m tiny.segment --compare bpe viterbi                   # tokenizer comparison table
python -m tiny.figures logs/ -o report/figures/                # every figure from logs
mvp-mcp                                                        # your MVP as an MCP server
```

**Required behavior**

- **Model and claim (ML/DL).**
    - Train a decoder-only transformer with 2–4 blocks in PyTorch on a small corpus, ideally text from your product's domain.
    - Copy one trained block's weights into a NumPy decoder block (llm-agent-reads' or your Week 8 version). Its outputs match PyTorch to within float tolerance.
    - Reproduce one published claim across at least 3 seeds. Suggested: post-norm blocks need learning-rate warmup and pre-norm blocks often do not (Xiong et al., 2020, "On Layer Normalization in the Transformer Architecture", [arXiv:2002.04745](https://arxiv.org/abs/2002.04745)). You may choose another claim if you can test it at this scale.
- **Segmentation (DSA).**
    - Implement a Viterbi segmenter (Word Break with scores) over your Week 7 vocabulary, with piece scores from corpus frequencies.
    - Compare it with your BPE on tokens per character and on a few hand-picked words. One table in the report.
- **MCP server (AI agents).**
    - Your MVP's core feature as an MCP server with at least two tools and one resource.
    - Input validation, and failures returned as tool execution errors.
    - A client script that lists the tools and calls each one.
- **Observability (Software dev).**
    - Structured logs for the training run and the MVP, as described above.
    - A usage summary script that reads the MVP logs and prints users, sessions, and completed tasks.

**Stretch**

- Repeat the claim with RoPE instead of learned absolute positions, and report whether the result holds.
- Publish the MCP server so others can install it, with a README a stranger can follow.

## Month 2 milestone

Day 60 is the milestone. You need two things: a technical report and an MVP that 10 people have used.

**Communication: an arXiv-style technical report.**

- Sections: abstract, introduction, related work, method, experimental setup, results, limitations, and a reproducibility statement with the exact commands.
- Results show every seed, not just the best one, with the spread across seeds.
- Every number in the report comes from a log file in the repo, through a script.
- State plainly what did not reproduce. A negative result, honestly measured, is a good report.
- Write it in LaTeX or Markdown, and publish the PDF and source on GitHub.
- By Day 60, have the full draft with results. Publish the final version on Day 63, after the defense.

**Users: the MVP is used by 10 people.**

- Define "used" before you count. For example: completed the core task at least once, on their own.
- Count from your logs, not from memory.
- Collect one quote from each user, with their permission to share it.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 57 | "A seed argument that seeds half the model." Show two `DecoderBlock`s with identical attention weights. |
| 58 | "Greedy change-making fails with coins 1, 3, 4." Then: the same recurrence is a Bellman equation. |
| 59 | "Word Break is how a Unigram tokenizer picks its tokens." Link SentencePiece's `Lattice::Viterbi`. |
| 60 | "My model's loss at step 0 should be ln(V). It wasn't." What the gap revealed. |
| 61 | "git's patience diff is Longest Increasing Subsequence." Link `xdiff/xpatience.c`. |
| 62 | Your reproduction result, with the plot across seeds. |
| 63 | The published report, and the 10-user count from your logs. |

**This week's write-up:** the Month 2 arXiv-style technical report, published on GitHub with its source, logs, and figure scripts.

## Value-creation step

Get all 10 people you recruited using the MVP by Day 60. For each one, record from your logs how many times they used it and whether they completed the core task, and collect one quote in their own words. Note the two users who used it most. You will ask them a harder question in [Week 10](week_10.md).
