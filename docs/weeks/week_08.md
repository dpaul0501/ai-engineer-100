# Week 8 (Days 50–56): Search Trees, Attention, and Agents That Plan

## This week

- **You will learn:**
    - **DSA:** how backtracking explores a tree of choices and prunes the branches that cannot succeed.
    - **ML/DL:** how positional encodings inject order, and how attention mixes tokens by relevance.
    - **AI agents:** how agents plan, and when splitting work across several agents helps or hurts.
    - **Software dev:** how to sketch a system before building it: components, data flow, and back-of-envelope numbers.
- **Goal:** by Sunday you can write batched, masked, multi-head attention with RoPE from a blank file, match PyTorch to within float tolerance, and defend every line.
- **Weekly build:** attention from scratch, tested against a reference on every small mask pattern, with a multi-agent mutation tester that checks whether your tests catch planted bugs. See [Weekly build](#weekly-build).
- **AI rule (Days 21–60):** AI pairs, you prove. AI may write code. Tests come first, and you must explain every line it wrote. This week the bar is higher: the build is "defended line by line", so write the attention core yourself and let AI write only the test harness.

## DSA: backtracking

**Read first:** [Tree of Thoughts: beam search over reasoning](https://mldsa.github.io/docs/part1/chapter03-dp/#8-tree-of-thoughts-beam-search-over-reasoning) in the DP chapter of *From LeetCode to Learning Systems*, and the [agents chapter](https://mldsa.github.io/docs/ml-spine/chapter-agents/).

**Focus.** Backtracking walks a tree of partial solutions. At each node you choose, recurse, then undo the choice. The tree is exponential, so the skill is not writing the recursion. It is pruning: rejecting a partial solution as soon as it cannot lead to a valid one. Every problem this week has the same shape: state, choices, a validity check, and an undo.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 50 | [78. Subsets](https://leetcode.com/problems/subsets/) | Include or exclude each element |
| 50 | [46. Permutations](https://leetcode.com/problems/permutations/) | Choose an unused element, then undo |
| 51 | [39. Combination Sum](https://leetcode.com/problems/combination-sum/) | Sort, then prune when the sum overshoots |
| 52 | [22. Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) | Only extend valid prefixes |
| 53 | [79. Word Search](https://leetcode.com/problems/word-search/) | Mark visited, recurse, unmark |
| 53 | [131. Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning/) | Cut points as choices |
| 54 | [51. N-Queens](https://leetcode.com/problems/n-queens/) | One set per constraint, checked in O(1) |

### Depth ladder: Combination Sum

1. **Solve.** Sort the candidates. Recurse with a start index so each combination is built in non-decreasing order, which avoids duplicates. Stop a branch as soon as the running sum exceeds the target. Write tests for the given examples first, and one where no combination exists.
2. **Vary.**
    - Each number may be used once, and the input has duplicates (LeetCode 40, Combination Sum II). How do you skip equal siblings without skipping valid repeats?
    - Exactly k numbers from 1–9 (LeetCode 216, Combination Sum III).
    - Count the combinations instead of listing them, where order matters (LeetCode 377, Combination Sum IV). Backtracking times out. Why does this become dynamic programming in Week 9?
3. **Break the assumptions.**
    - **Unsorted input.** Remove the sort and keep the `break` on overshoot. Which valid answers disappear?
    - **A zero in the candidates.** The recursion never terminates. The problem forbids it; your code should reject it, not hang.
    - **Negative numbers.** Pruning by "sum exceeds target" is now wrong. What can you still prune?
    - **Recursion depth.** Target 5000 with candidate 1. Python's default recursion limit is 1000. Rewrite with an explicit stack.
4. **ML lens.**
    - Hyperparameter grid search enumerates a product of choices, with no pruning at all. Early-stopping schedulers such as successive halving are pruning: they abandon a branch once it looks hopeless.
    - Agent planning is search over action sequences. A plan step that fails validation is a branch you backtrack from. See the AI agents section below.
5. **In the wild.**
    - scikit-learn's `ParameterGrid` in `sklearn/model_selection/_search.py`. Iteration uses `itertools.product`. Read `__getitem__`: it computes the i-th combination directly, without enumerating the ones before it. How?
    - pip's dependency resolver is `resolvelib`. Read `src/resolvelib/resolvers/resolution.py` and find `_backjump`. Backjumping skips several levels of the tree at once. Why is plain one-level backtracking too slow for dependency graphs?

### Depth ladder: Generate Parentheses

1. **Solve.** Add `(` while opens remain. Add `)` only while it would not close more than it opens. Every leaf is valid, so there is nothing to filter at the end. State the count of results: the n-th Catalan number.
2. **Vary.**
    - Two bracket types, `()` and `[]`. What extra state does the validity check need? (A stack, as in Week 3.)
    - Generate only the k-th string in lexicographic order, without generating the rest.
3. **Break the assumptions.**
    - Generate all 2ⁿ strings and filter afterwards. Time both versions at n = 12. How much of the tree did the filter version visit?
    - n = 20 returns about 6.5 million strings. Yield them from a generator instead of building a list, and measure the memory difference.
4. **ML lens.** This is constrained decoding. When an LLM must emit valid JSON, the decoder masks every token that would make the prefix invalid, exactly as you refuse `)` when no `(` is open. The mask is applied to the logits before sampling, with `-inf` in the blocked positions. You will see the same `-inf` trick in attention masking below and in sampling in [Week 10](week_10.md).
5. **In the wild.** CPython's regex engine is a backtracking matcher: `Modules/_sre/sre_lib.h`. Time `re.match(r"(a+)+$", "a" * n + "b")` for n = 16, 18, 20, 22. The time roughly doubles with each extra `a`. That is catastrophic backtracking. Python 3.11 added atomic groups `(?>...)` and possessive quantifiers `a++`. Rewrite the pattern with one and time it again.

### Depth ladder: N-Queens

1. **Solve.** Place one queen per row. Keep three sets: columns, `r - c` diagonals, and `r + c` anti-diagonals. Each check is O(1).
2. **Vary.**
    - Count solutions only (LeetCode 52, N-Queens II). Replace the sets with bitmasks.
    - Sudoku (LeetCode 37). Pick the cell with the fewest legal values first. Measure how much smaller the tree gets.
3. **Break the assumptions.** Replace the three sets with a scan of the board for every placement. At n = 10, how many times slower is it, and why?
4. **ML lens.** "Most constrained variable first" is a search heuristic. Planners and schedulers use the same rule. A learned heuristic that scores which branch to try first is the idea behind game-tree search with a value network.
5. **In the wild.** `itertools.permutations` in CPython's `Modules/itertoolsmodule.c` produces permutations without recursion, using an index array and a cycles array. Trace it for n = 3 and compare it with your recursive version.

### Shorter ladders: Subsets, Permutations, Word Search, Palindrome Partitioning

- **Subsets.** Vary: input with duplicates (LeetCode 90, Subsets II). Break: n = 30 has about a billion subsets. ML lens: feature selection over all subsets is exponential, which is why greedy forward selection exists. In the wild: `itertools.combinations` in the same C file.
- **Permutations.** Vary: duplicates in the input (LeetCode 47). Break: a `used` list that you forget to reset on the way back. Write the test that catches it.
- **Word Search.** Vary: many words at once (LeetCode 212, Word Search II) with the trie from Week 5. Break: forget to unmark a cell after recursion. Which grid exposes it?
- **Palindrome Partitioning.** ML lens: a string has exponentially many tokenizations. Unigram tokenizers score all of them, and BPE-dropout samples among them. In Week 9 you count and score them with DP instead of listing them.

## ML/DL: positional encoding and attention

### Learn

- Why attention without positions is permutation-equivariant: shuffle the tokens and the outputs shuffle the same way
- Sinusoidal absolute encodings, frequencies, and phase ([position chapter](https://dpaul0501.github.io/llm-agent-reads/part_02_position_encoding/))
- RoPE: rotate query and key pairs so the score depends only on the relative offset
- Queries, keys, values, the scaled score, softmax over the key axis, and the weighted sum ([attention chapter](https://dpaul0501.github.io/llm-agent-reads/part_03_attention/))
- Causal and padding masks, multi-head split and combine, and the O(T²) cost

### Read the code

**1. This repo's [`positional_encoding.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/positional_encoding.py) and [`rope.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/rope.py).** Answer in your log:

- `apply_rope` has separate branches for 2-D and 3-D input and raises on 4-D. What shape do queries have inside multi-head attention? Rewrite it with one broadcasting path.
- `rotate_half` pairs dimensions `(0, 1), (2, 3), ...`. Now read `rotate_half` in Hugging Face's `models/llama/modeling_llama.py`. It pairs `(0, d/2), (1, d/2 + 1), ...`. Both are RoPE. Why can't you load weights trained with one into the other? Find `permute` in `convert_llama_weights_to_hf.py` in the same folder.
- `test_rope_relative_offset_and_pair_rotation` shifts both positions by 7 and checks the score is unchanged. Is that enough to prove the rotation is correct? Write one implementation that is wrong but passes it.

**2. This repo's [`math_utils.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/math_utils.py), [`self_attention.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/self_attention.py), and [`multi_head_attention.py`](https://github.com/dpaul0501/llm-agent-reads/blob/main/llm_foundations/multi_head_attention.py).** You read the first two in Week 1. Now fix them.

- `scaled_dot_product` uses `k.transpose(0, 2, 1)`, so it accepts only 3-D tensors. That is why `MultiHeadAttention` loops over heads in Python. Change it to `np.swapaxes(k, -1, -2)` and remove the loop.
- The mask fills blocked scores with `-1e9`. A row with every key masked becomes a uniform average of `V`: finite, plausible, and wrong. Decide what your version returns for that row, and test it.
- `softmax` casts to float64. Your version should keep the input dtype. Then check that float32 still sums to 1 within tolerance.

**3. PyTorch's `scaled_dot_product_attention`.** The docstring in `torch/nn/functional.py` contains a short Python reference implementation. Read it line by line.

- It fills blocked positions with `float("-inf")`, not `-1e9`. What does a fully masked row produce there?
- With `is_causal=True` it builds `tril` on an L×S matrix. When L = 1 and S = 10, as in cached decoding, which keys can the single query see? Week 10 depends on the answer.
- The boolean mask means "True participates". `nn.MultiheadAttention`'s `key_padding_mask` means "True is masked out". The docstring warns about this. Write the test that would catch a flipped mask.

**4. nanoGPT's `CausalSelfAttention`** in [`model.py`](https://github.com/karpathy/nanoGPT/blob/master/model.py). One `c_attn` linear layer produces Q, K, and V at once, then `split` and `view` separate the heads. Compare the shapes with your version, and note when it calls PyTorch's fused kernel.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `apply_rope` (llm-agent-reads) | `base` | `10000.0` | How fast the rotation frequencies fall across dimension pairs | Small base: even the slow pairs wrap within the context, so distant positions alias. Very large base: most pairs barely rotate, so nearby positions look the same |
| `apply_rope` | `positions` | `None` (0 to T−1) | The position each row is rotated by | During cached decoding the new token is at position T, not 0. Passing the default gives finite, wrong scores |
| `sinusoidal_positions` | `d_model` | required, must be even | Number of sine/cosine pairs | Odd sizes raise. Tiny sizes give too few frequencies to tell long positions apart |
| `scaled_dot_product_attention` (llm-agent-reads) | `mask` | `None` | Which keys each query may see | `None` in a decoder lets tokens see the future: training loss looks great, generation is garbage. A fully masked row silently averages `V` |
| `MultiHeadAttention` (llm-agent-reads) | `num_heads` | required, must divide `d_model` | How many separate attention patterns | One head: one pattern per token. `num_heads = d_model`: head size 1, and each score is a product of two scalars |
| `F.scaled_dot_product_attention` | `is_causal` | `False` | Apply a causal mask internally | Mask is aligned top-left, so with fewer queries than keys it blocks almost everything |
| `F.scaled_dot_product_attention` | `scale` | `None` (1/√d) | Score temperature | Too large: softmax saturates to one-hot and gradients vanish. Too small: every row is nearly uniform |
| `F.scaled_dot_product_attention` | `dropout_p` | `0.0` | Dropout on attention weights | It is applied whenever nonzero, even in eval. You must pass `0.0` at inference yourself |
| `nn.MultiheadAttention` | `batch_first` | `False` | Whether input is (T, B, E) or (B, T, E) | Pass (B, T, E) with the default and batch is treated as time. No error when shapes are compatible |
| `nn.MultiheadAttention` | `dropout` | `0.0` | Dropout on attention weights | High values during training make attention patterns noisy and slow learning |

### Break it

- **Permutation test.** Shuffle the input tokens of your attention, with no positions and no mask. Show the outputs are the same rows, shuffled. Add RoPE and show they are not.
- **The silent average.** Build a padding mask where one sequence is all padding. Run llm-agent-reads' version and PyTorch's. One gives a plausible vector, the other gives NaN. Which failure would you rather debug in production?
- **Scale.** Remove the `1/√d` scale and set `d = 512` with random unit-variance inputs. Plot the maximum attention weight per row. Explain the result in one sentence.

## AI agents: planning and multi-agent systems

### Learn

- ReAct-style loops (decide one step at a time) versus plan-and-execute (write a plan first, then run it)
- Re-planning: when to revise the plan after a step fails
- Orchestrator–worker and handoff patterns, and what state each agent sees
- The [agents chapter](https://mldsa.github.io/docs/ml-spine/chapter-agents/) in *From LeetCode to Learning Systems*
- The costs of multi-agent: more calls, more tokens, and information lost at every handoff
- Planning as search: Tree of Thoughts explores several reasoning paths with backtracking (Yao et al., 2023, [arXiv:2305.10601](https://arxiv.org/abs/2305.10601))

### Experiment

Take 10 tasks from your product's domain that need 3 or more tool calls each. Run each task three ways:

- a single ReAct-style agent
- a planner that writes the full plan, then an executor that follows it
- an orchestrator that hands sub-tasks to two worker agents

For each, measure success rate (graded against expected answers you write first), total tokens, number of LLM calls, and wall time. Put the results in one table. Multi-agent is not automatically better. Find the task type where it wins, and the one where it loses.

### Break it

- Give the planner a task that needs a tool it does not have. Does it invent the tool, or say it cannot?
- Make step 2 fail every time. Count how many times the agent re-plans before it stops. If it never stops, add a budget.
- Pass only a summary between agents. Find a task where the worker fails because the detail it needed was summarized away.

### ML lens

A plan is a path through a tree of actions, the same tree as this week's DSA. A planner that validates each step and backs up on failure is backtracking. One that keeps the best few partial plans is beam search, which you implement for decoding in Week 10.

### In the wild

Pick one open-source multi-agent framework, such as LangGraph, AutoGen, or CrewAI. Find the code that decides which agent runs next. Is it a fixed graph, a rule, or an LLM call? What happens when that decision is wrong?

## Software dev: system design basics

### Learn

- Components and data flow: client, API, workers, storage, and the LLM provider as an external dependency
- Back-of-envelope estimates: requests per day, tokens per request, latency per call, and storage per user
- Latency vs throughput, and where caching, queues, and timeouts belong
- Stateless services, and where state must live
- A one-page design doc: goal, non-goals, diagram, the main risk, and what changes at 10× users

### Apply it

Write a one-page design doc for the MVP v0 you ship this week (see [Value-creation step](#value-creation-step)). Include a diagram, an estimate of p50 latency and cost per request, and the single component most likely to fail.

### Verify

Estimates are hypotheses. Run 50 real requests through MVP v0 and compare measured p50 latency and cost per request with your estimate. Record where you were off by more than 2×, and why.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 50 | Subsets, Permutations | Permutation equivariance; sinusoidal encodings | ReAct vs plan-and-execute | Components and data flow |
| 51 | Combination Sum | RoPE; read `rope.py` and Llama's `rotate_half` | Planner–executor on 10 tasks | Back-of-envelope estimates |
| 52 | Generate Parentheses | Q, K, V and scaled scores; read `math_utils.py` | Re-planning and budgets | Design doc for MVP v0 |
| 53 | Word Search, Palindrome Partitioning | Masks; read PyTorch's reference SDPA | Orchestrator–worker; measure handoff loss | Caching, queues, timeouts |
| 54 | N-Queens | Multi-head attention; read nanoGPT's attention | Compare the three setups in one table | Measure MVP v0 against the estimate |
| 55 | **Weekly build** | | | |
| 56 | **Defend and publish** | | | |

## Weekly build

### Attention from scratch, defended line by line

```bash
python -m attn.check                  # compare against PyTorch on random inputs
python -m attn.exhaustive --max-t 4   # every small mask pattern
python -m attn.mutate --n 10          # planted bugs: how many do your tests catch?
```

**Required behavior**

- **Attention (ML/DL).**
    - Write `attention(q, k, v, mask=None)` in NumPy for any number of leading dimensions, including (batch, heads, seq, dim).
    - The softmax is stable and keeps the input dtype.
    - Decide and document what a fully masked row returns. It must not be a silent average.
    - Write `MultiHeadAttention` with no Python loop over heads, and RoPE applied to queries and keys with an explicit `positions` argument.
    - Match `torch.nn.functional.scaled_dot_product_attention` on random inputs, in float64 and float32.
- **Exhaustive small cases (DSA).**
    - Use backtracking to enumerate every boolean mask for T ≤ 4 that your API accepts, pruning patterns that are invalid by your own rules.
    - Run your attention and the reference on each one, and report every mismatch.
- **Mutation tester (AI agents).**
    - A planner agent reads your attention file and proposes 10 small bugs: a wrong axis, a missing scale, a flipped mask.
    - A worker agent applies each bug to a scratch copy, never your working tree.
    - Your test suite runs against each mutant. Report how many your tests caught, and add tests for the ones they missed.
- **Engineering (Software dev).**
    - A README with a short design note: module layout, data flow through the three commands, and what you would change to run this on a GPU.
    - pytest suite, type hints, and a log of which parts AI wrote.

**Stretch**

- Add grouped-query attention: fewer key/value heads than query heads. Match PyTorch with `enable_gqa=True`.
- Plot attention weights for a short sentence with and without RoPE.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 50 | "Attention without positions cannot tell 'dog bites man' from 'man bites dog'." Show the permutation test. |
| 51 | "Two RoPE implementations that are both correct and not compatible." Interleaved vs half-split, with the Llama `permute` line. |
| 52 | "Generate Parentheses is constrained decoding." The `-inf` mask in both. |
| 53 | "A fully masked row: one library returns NaN, another returns a plausible average." Show both. |
| 54 | "I ran the same 10 tasks with one agent and with three. Here is the table." |
| 55 | Your attention module matching PyTorch, with the mutation score. |
| 56 | What the defense exposed, and what you fixed. |

**This week's write-up:** a Medium article on the hardest line in your attention code, with the test that proves it and links to the real source files you compared against.

## Value-creation step

Ship MVP v0 to the first 3 of the 10 people you recruited in Week 7. Watch each one use it, live or on a screen share, without helping. Write down where each person got stuck, what they tried first, and the one thing they asked for. Fix the worst problem before Week 9, when all 10 get access.
