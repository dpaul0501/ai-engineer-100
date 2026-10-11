# Week 13 (Days 85–91): Bits, Research, and the Month 3 Milestone

## This week

- **You will learn:**
    - **DSA:** how integers are stored as bits, and how XOR, masks, and repeated squaring solve problems in O(1) space or O(log n) time.
    - **ML/DL:** how to do research: find what is new, survey related work, reproduce a baseline, and write it up.
    - **AI agents:** how to put the agent at the center of your capstone, with an eval gate on every change.
    - **Software dev:** how to write docs a stranger can follow, and how to open-source part of your work.
- **Goal:** by Day 90 you have submitted a workshop paper or filed a provisional patent, and you have a first paying user or a signed letter of intent.
- **Weekly build:** the Month 3 milestone, plus the capstone agent behind an eval gate and one open-sourced component. See [Weekly build](#weekly-build).
- **AI rule (Days 61–100):** AI agents, you review. Agents write code. You write the spec, review the PR, run the evals, and reject bad output.

!!! warning "Patent or paper: decide on Day 85"
    In India and most other countries a patent requires absolute novelty. A public post, repo, talk, or preprint that describes your invention can make it unpatentable. If you choose the patent route, file a provisional application **before** you publish anything about the idea, and talk to a registered patent agent. This page is not legal advice. If you go for a patent, this week's daily posts cover the other parts of your work: DSA findings, deployment, evaluation, docs. See the [publishing and patents warning](../how-it-works.md).

## DSA: math and bit manipulation

**Read first:** the XOR and bitsets rows of [DSA to ML](https://mldsa.github.io/docs/index-dsa-to-ml/) in *From LeetCode to Learning Systems*.

**Focus.** An integer is a row of bits, and a few operations do a lot. `x ^ x == 0` cancels pairs. `x & (x - 1)` clears the lowest set bit. `x & -x` isolates it. A mask `& 0xFFFFFFFF` pins a value to 32 bits. Python makes this tricky in one way: its integers have no fixed width, so negative numbers behave as if they had infinitely many leading 1 bits. Many textbook bit tricks loop forever or return the wrong answer in Python unless you mask.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 85 | [136. Single Number](https://leetcode.com/problems/single-number/) | XOR cancels pairs |
| 85 | [268. Missing Number](https://leetcode.com/problems/missing-number/) | XOR indices with values |
| 86 | [191. Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/) | `n & (n - 1)` clears one bit |
| 86 | [338. Counting Bits](https://leetcode.com/problems/counting-bits/) | `bits[i] = bits[i >> 1] + (i & 1)` |
| 87 | [190. Reverse Bits](https://leetcode.com/problems/reverse-bits/) | Shift out, shift in, 32 times |
| 88 | [371. Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/) | XOR is sum without carry; AND is the carry |
| 89 | [50. Pow(x, n)](https://leetcode.com/problems/powx-n/) | Repeated squaring |

### Depth ladder: Single Number

1. **Solve.** XOR every element. Pairs cancel because `a ^ a == 0` and XOR is associative and commutative. What remains is the single one. O(n) time, O(1) space. Write tests for the given examples first.
2. **Vary.**
    - Every other element appears three times (LeetCode 137, Single Number II). Count each bit position modulo 3.
    - Two elements appear once (LeetCode 260, Single Number III). XOR everything, then split the numbers by the lowest set bit of the result, `x & -x`.
    - One number is missing and one is duplicated. Combine XOR with a sum.
3. **Break the assumptions.**
    - The input breaks the promise: one element appears three times. XOR returns a wrong number and raises nothing. Your code trusts its input. Where should the check go?
    - Floats. `1.5 ^ 1.5` is a `TypeError`. You can XOR the bit patterns with `struct`, but then `0.0` and `-0.0` are different, and NaN has many bit patterns.
    - Data arrives over a network in pieces. XOR of all pieces detects that something is missing, but not which piece.
4. **ML lens.**
    - **Order-independent hashing.** XOR combines hashes without caring about order, which is useful for hashing a set of features. But `h ^ h == 0`, so duplicates cancel and some different sets collide. Python's `frozenset` hash avoids this: read `frozenset_hash_impl` and `_shuffle_bits` in CPython `Objects/setobject.c`. Why does it scramble each entry's hash before XOR?
    - **Checksums** on data shards catch silent corruption in a training pipeline.
5. **In the wild.**
    - CPython `Objects/setobject.c`, as above. Compare `hash(frozenset({1, 2}))` with `hash(1) ^ hash(2)`.
    - RAID 5 stores the XOR of the data blocks as parity. Lose one disk, XOR the rest, and you get it back. That is Single Number with blocks.

### Depth ladder: Number of 1 Bits

1. **Solve.** Loop `n &= n - 1` and count the steps. Each step clears the lowest set bit, so the loop runs once per 1 bit. Also try `bin(n).count("1")` and `n.bit_count()` (Python 3.10+). Time all three.
2. **Vary.**
    - Counting Bits (338): every number from 0 to n in O(n) with DP over bits.
    - Hamming distance (LeetCode 461): `(x ^ y).bit_count()`.
    - Total Hamming distance over all pairs (LeetCode 477). Count per bit position in O(32n), not O(n²).
3. **Break the assumptions.**
    - **Negative numbers in Python.** Run `n = -1; while n: n &= n - 1`. It never ends: `-1 & -2` is `-2`, then `-4`, then `-8`, forever. Python integers have no fixed width.
    - `(-1).bit_count()` returns 1. It counts the bits of the absolute value. A 32-bit two's-complement answer is 32: use `(n & 0xFFFFFFFF).bit_count()`.
    - `bin(-5)` is `'-0b101'`. Counting `"1"` characters gives 2, not the two's-complement count.
4. **ML lens.**
    - **Binary embeddings.** Store each embedding as bits (1 if the value is positive), and the distance becomes popcount of XOR. That is 32× less memory than float32. FAISS supports it with `IndexBinaryFlat` (`faiss/IndexBinaryFlat.h`). Binarize your Week 6 RAG embeddings and measure how much recall@10 drops.
    - **Bitsets** for fast set operations, as in Week 1's Bloom filter.
5. **In the wild.**
    - CPython `Objects/longobject.c`: `int_bit_count_impl` loops over the integer's internal digits and sums `popcount_digit` for each. `popcount_digit` calls `_Py_popcount32`, which uses `__builtin_popcount` where the compiler has it (`Include/internal/pycore_bitutils.h`). Why does CPython store integers in 30-bit digits, not 32?

### Depth ladder: Pow(x, n)

1. **Solve.** Repeated squaring. Walk the bits of `n`. Square the base each step, and multiply it into the result when the bit is 1. Handle negative `n` with `1 / x^(-n)`. O(log n) multiplications.
2. **Vary.**
    - Modular power, `pow(a, b, m)`, for hashing and cryptography.
    - **Matrix power** for linear recurrences: the n-th Fibonacci number in O(log n). Compare with the O(n) DP from Week 9.
    - Replace `+` and `×` with `min` and `+` (the min-plus semiring), and matrix power computes shortest paths of at most k edges. That links to [Week 12](week_12.md).
3. **Break the assumptions.**
    - `n = -2³¹`. Negating it overflows a 32-bit int in Java or C++. Python is fine. Why?
    - `x = 0` and `n < 0`. What should happen, and what does your code do?
    - `x = 1.00001`, `n = 10⁹`. Compare your answer with `math.pow` and with `math.exp(n * math.log(x))`. Where does rounding creep in?
    - Overflow to `inf`, and underflow to `0.0`.
4. **ML lens.**
    - **Learning-rate schedules.** Exponential decay is `lr * gamma ** step`. At large `step` it underflows to zero.
    - **RoPE frequencies** are `base ** (-2i / d)`, the powers you computed in Week 8 ([position encoding chapter](https://dpaul0501.github.io/llm-agent-reads/part_02_position_encoding/)).
    - Linear recurrences, as in state-space models, are matrix powers.
5. **In the wild.**
    - CPython `long_pow` in `Objects/longobject.c`. For moderate exponents it uses left-to-right binary exponentiation; the comment cites HAC Algorithm 14.79. For very large exponents it switches to a k-ary method with a precomputed table. Why would left-to-right be faster than the right-to-left loop you wrote?

### Day 85 extra: Missing Number

- **Vary:** XOR all indices and values, or use the sum formula `n(n+1)/2 − sum`.
- **Break:** the sum overflows a 32-bit int for large `n`. XOR does not.
- **ML lens:** a row count or XOR checksum before and after each pipeline step catches dropped rows.

### Day 86 extra: Counting Bits

- **Vary:** `bits[i] = bits[i & (i - 1)] + 1` is another valid recurrence. Prove both.
- **ML lens:** a lookup table computed once and reused is how fast popcount worked before CPUs had an instruction for it.

### Day 87 extra: Reverse Bits

- **Break:** in Python, shifting left never drops bits. Without a final `& 0xFFFFFFFF`, or a loop of exactly 32 steps, your result grows past 32 bits.
- **ML lens:** the FFT reorders its input by bit-reversed index.

### Day 88 extra: Sum of Two Integers

- **Vary:** `a ^ b` is the sum without carries. `(a & b) << 1` is the carry. Repeat until the carry is 0.
- **Break:** in Python, with a negative input, the carry never reaches 0 and the loop never ends. Mask every step with `0xFFFFFFFF`, then convert back: if the result is above `0x7FFFFFFF`, it is negative.
- **ML lens:** low-precision formats are bit layouts. bfloat16 is the top 16 bits of a float32. Use `struct.pack("<f", x)` to see the bits of a float32 and truncate it to bfloat16 by hand. How much precision is lost?

## ML/DL: research week

### Learn

- **What counts as new.** A new problem, a new method, a new dataset or evaluation, or a careful negative result. "It works on my product" becomes a contribution only when you show what it does that existing methods do not
- **Related work.** Search arXiv, Semantic Scholar, and the reference lists of the closest papers. Walk the citation graph both ways: what a key paper cites, and what cites it. Keep a table: paper, problem, method, dataset, result, how yours differs
- **Reproducing a baseline.** Same data, same split, same metric, same preprocessing. Report the number you get next to the number the paper reports, with your seeds and an interval
- **Ablations.** Remove one piece at a time and show what each piece contributes
- **Writing.** Abstract, introduction (problem, gap, contributions as bullets), method, experiments, limitations. Write the contribution bullets first
- **Workshop logistics.** Page limits, templates, double-blind anonymity rules, and deadlines. Read the call for papers before you write a word

### Read the code

**1. The official repo of your baseline paper.** Find and write down:

- Where the random seed is set, and what it covers.
- Where the train/test split is made, or loaded.
- Which metric implementation it uses, and whether it matches the paper's definition.
- Any preprocessing that the paper does not mention.

**2. Hugging Face `transformers`, `set_seed` and `enable_full_determinism` in [`src/transformers/trainer_utils.py`](https://github.com/huggingface/transformers/blob/main/src/transformers/trainer_utils.py).**

- `set_seed` calls `random.seed`, `np.random.seed`, and `torch.manual_seed`. Does it seed a generator you created with `np.random.default_rng()`? Test it.
- `enable_full_determinism` sets `CUBLAS_WORKSPACE_CONFIG` to `":16:8"`, calls `torch.use_deterministic_algorithms(True, warn_only=warn_only)`, and turns off `cudnn.benchmark`. What does each one fix, and what does it cost in speed?

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `transformers.set_seed` | `deterministic` | `False` | Also call `torch.use_deterministic_algorithms(True)` | `False`: same seed, different results on GPU. `True`: slower, and some ops raise errors |
| `enable_full_determinism` | `warn_only` | `False` | Raise or warn on non-deterministic ops | `True` hides the op that breaks reproducibility |
| `torch.use_deterministic_algorithms` | `mode` | required | Deterministic kernels only | Some ops have no deterministic version and raise |
| `torch.backends.cudnn.benchmark` | (flag) | `False` | Let cuDNN pick the fastest kernel by timing | `True` can pick different kernels per run, so results drift |
| `scipy.stats.ttest_rel` | `alternative` | `'two-sided'` | Which difference you test for | Choosing `'greater'` after seeing the data inflates false positives |
| `scipy.stats.wilcoxon` | `zero_method` | `'wilcox'` | How tied (zero-difference) pairs are handled | Many ties, as with exact-match scores, change the result a lot |
| `scipy.stats.bootstrap` | `n_resamples` | `9999` | Resamples for the interval | Too few in a paper: the interval changes when a reviewer reruns it |

### Break it

Reproduce your baseline with five seeds. Then change one preprocessing detail the paper does not mention, such as lowercasing or the maximum sequence length. Report how far the number moves. If it moves more than the gain you plan to claim, your claim needs more evidence.

## AI agents: the agent behind the capstone

### Learn

- The agent's job in one paragraph: what it does, what it must never do, when it hands off to a human
- The smallest tool set that does the job. Each tool with a clear schema, a timeout, and a permission scope
- Traces for every run: inputs, tool calls, outputs, cost, latency, outcome
- **An eval gate in CI.** A golden set built from real (consented, anonymized) user tasks. A PR that drops the score beyond your threshold does not merge
- LLM output varies run to run. Run the eval set several times and gate on the interval, not one number
- Human handoff: when the agent is unsure or out of scope, it says so and routes the user to a person

### Experiment

Build an eval set of 50 real user tasks from your traces. For each task, write what success looks like. Measure task success, cost per task, p95 latency, and handoff rate. Run the set three times and report the mean and spread. Then wire it into CI so every PR runs it, and record how many agent PRs the gate blocks this week and why.

### Break it

- Replay a session where a tool result contains an instruction hidden by an attacker (Week 10). Does the agent follow it?
- Make a tool return an empty result. Does the agent say so, or invent an answer?
- Ask something just outside the agent's scope. Does it hand off, or improvise?

### ML lens

Your eval gate is the experiment design from [Week 12](week_12.md) run on every PR. A failure taxonomy (wrong tool, wrong argument, hallucinated result, missed handoff) is a confusion matrix for agent behavior. Count it.

### In the wild

- MCP reference servers: `src/filesystem/path-validation.ts` in [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers). How does it decide whether a requested path is inside an allowed directory, and what does normalizing do to `..`? Then search `src/filesystem/lib.ts` for `realpath`: why is a symlink checked separately? Compare it with how your agent's tools limit what they can touch.
- OpenTelemetry GenAI conventions, `docs/gen-ai/gen-ai-agent-spans.md` in [open-telemetry/semantic-conventions-genai](https://github.com/open-telemetry/semantic-conventions-genai). Which fields would you need to add to your traces to follow it?

## Software dev: docs and open source

### Learn

- Four kinds of docs: tutorials, how-to guides, reference, and explanation. Each has a different reader
- A README that answers what, why, and how in five minutes: install, one example, expected output, limits
- Docstrings that the docs site builds from. Examples in docstrings that run as tests (`pytest --doctest-modules`)
- `LICENSE`, `CONTRIBUTING.md`, a code of conduct, issue templates, and "good first issue" labels
- Semantic versioning, a `CHANGELOG`, and tagged releases
- **Licenses and patents.** Some licenses include a patent grant (Apache-2.0 does; MIT does not mention patents). Open-sourcing the inventive part of your work is public disclosure. If you are on the patent route, open-source only components that are not part of the invention, and ask your patent agent before you choose a license

### Apply it

Open-source one component of your capstone that is useful on its own: your eval harness, a CLI, a data cleaner, or the replay tool from Week 12.

- Its own repo, with a license, README, docs site, and CI.
- A coding agent may draft the docs from your outline. You review every page against the code.
- Tag a `0.1.0` release.

### Verify

- Ask another participant to follow your quickstart on a clean machine while you watch without helping. Time it. Write down every place they got stuck, and fix each one.
- Doctests run in CI. Break one example on purpose and show that CI fails.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 85 | Single Number, Missing Number | Novelty; **decide: paper or patent** | The agent's job, tools, and limits | Four kinds of docs; README outline |
| 86 | Number of 1 Bits, Counting Bits | Related-work table; walk the citation graph | Build the 50-task eval set | Docstrings and doctests |
| 87 | Reverse Bits | Reproduce the baseline with five seeds | Run the eval three times; failure taxonomy | License, CONTRIBUTING, issue templates |
| 88 | Sum of Two Integers | Ablations; write the contributions first | Eval gate in CI | Open-source one component; stranger test |
| 89 | Pow(x, n) | Full draft (paper) or draft spec with your patent agent | Break it: injection, empty tool, out of scope | Tag `0.1.0`; fix the stranger-test findings |
| 90 | **Month 3 milestone:** submit or file; first paying user or signed LOI | | | |
| 91 | **Defend and publish** | | | |

## Weekly build

### The agent behind the capstone

The capstone's agent, gated by evals, plus one open-sourced component, plus the Month 3 milestone below.

**Required behavior**

- **Agent (AI agents).**
    - A one-paragraph job description in the repo: what it does, what it must not do, when it hands off.
    - Every tool has a schema, a timeout, and a permission scope.
    - Traces for every run.
- **Eval gate (AI agents + ML/DL).**
    - 50 real tasks with success criteria, run in CI on every PR.
    - The gate uses an interval from repeated runs, not a single score.
    - A log of PRs the gate blocked, and why.
- **Bits where they fit (DSA).**
    - Use one of this week's ideas where it fits your product. Examples: a bitset for feature flags or permissions, a XOR checksum on exported data, or binary embeddings for a fast first-pass search.
- **Open source (Software dev).**
    - One component in its own repo with a license, README, docs, CI, and a `0.1.0` tag.
    - Stranger-test notes and the fixes they led to.
- **Research (ML/DL).**
    - The related-work table, the baseline reproduction with seeds, and the ablations, all in the repo (or kept private if on the patent route).

**Stretch**

- Run your eval set against two agent configurations and report which one wins, with a paired interval.
- Get one outside contributor to merge a PR into your open-source component.

## Month 3 milestone (Day 90)

| Milestone | Communication | Users and value |
|---|---|---|
| Day 90 | Workshop paper submission, or a provisional patent | First paying user, or a signed letter of intent |

### Communication: choose one route on Day 85

**Paper route.**

- Pick a workshop whose topic fits and whose deadline you can meet. Follow its template, page limit, and anonymity rules exactly.
- Required sections: problem, related work (from your table), method, the reproduced baseline, your results with intervals, ablations, limitations.
- If no workshop deadline falls near Day 90, finish the submission-ready PDF by Day 90 and record the target venue and its deadline in your repo.
- A mentor reads the full draft before you submit.

**Patent route.**

- Write an invention disclosure: the problem, what exists today, what your method does differently, and how it works, step by step, with diagrams. Include variations and alternatives.
- Work with a registered patent agent to file a **provisional** application. Ask them about deadlines for the complete application and about filing in other countries.
- **File before you publish anything about the idea.** That includes LinkedIn posts, the open-source component, a demo video, and a preprint.
- This week's daily posts cover the other parts of your work. Once the provisional is filed, ask your patent agent what you may then share.

### Users and value: first paying user or signed letter of intent

- Follow up on the proposals from [Week 11](week_11.md) and the retention data from [Week 12](week_12.md).
- A **paying user** pays any real amount for continued use.
- A **letter of intent** is a short signed document: who they are, the problem, what they will pay, and under what conditions (for example, when a success metric is met).
- If both candidates say no, write down why, in their words, and ask one more user from your list before Day 90.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day. **On the patent route, skip any post that touches the invention and pick a DSA, deployment, or docs finding instead.**

| Day | Post idea |
|---|---|
| 85 | "Why Python's `frozenset` hash doesn't just XOR." `_shuffle_bits` and the collisions it prevents. |
| 86 | "`while n: n &= n - 1` never ends in Python." Negative numbers and infinite leading ones. |
| 87 | "Same seed, different result." What `set_seed` covers, and what it does not. |
| 88 | "I reproduced a paper's baseline. Here is the gap." Your number vs theirs, with seeds. (Paper route only.) |
| 89 | "A stranger tried my README. They got stuck in 4 places." What you fixed. |
| 90 | The milestone: "Submitted to <workshop>" or "Filed a provisional" (without describing the invention), and your first paying user or LOI. |
| 91 | What the defense exposed, and what you fixed. |

**This week's write-up: the paper or the patent.** On the paper route, the submitted PDF, plus a public summary if the workshop's anonymity rules allow it. On the patent route, the filed provisional stays private. Publish a short post that you filed, without describing the invention.

## Value-creation step

Close your first paying user or signed letter of intent by Day 90. Record what they agreed to, the price, and the condition. Then ask them one question: "What would make you tell a colleague about this?" Their answer shapes the Day 100 launch.
