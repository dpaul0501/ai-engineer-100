# Week 5 (Days 29–35): Trees, Leakage, and Memory

## This week

- **You will learn:**
    - **DSA:** how trees carry ranges and prefixes, and how a trie turns many string lookups into one walk.
    - **ML/DL:** how decision trees split, how boosting adds trees, and how leakage makes a model look better than it is.
    - **AI agents:** how an agent remembers across turns and sessions when the context window is finite.
    - **Software dev:** how to model data in SQL, enforce it with constraints, and check queries with the query plan.
- **Goal:** by Sunday you can train a tree model on real tabular data, prove it does not leak, store everything in SQL, and talk to it through an agent that remembers what you told it yesterday.
- **Month 1 milestone (Day 30):** a technical blog post and a summary of your 5 user interviews. See [Month 1 milestone](#month-1-milestone).
- **Weekly build:** a leak-checked boosting model on a real dataset, stored in SQLite, with an agent that keeps memory across sessions. See [Weekly build](#weekly-build).
- **AI rule (Days 21–60):** AI pairs, you prove. AI may write code. Tests come first, and you must explain every line it wrote.

## Month 1 milestone

Due on **Day 30**. It is part of this week, not extra to it. The Day 30 row in the [Day by day](#day-by-day) table is lighter on purpose.

- **Technical blog post (Medium).**
    - Take one finding from Weeks 1–4 and go deep: one idea, with code, a measurement, and a link to a real source file.
    - Good topics: a seeded bug your tests missed, a gradient check that caught a wrong derivative, a tool-calling failure you reproduced.
    - A stranger should be able to rerun your code from the post.
- **Summary of 5 user interviews.**
    - One page. For each interview: who (role, not name), the context, and what they said in their own words.
    - Then the **top 3 pains**, ranked. For each: how many of the 5 people raised it, one direct quote, and what they do about it today.
    - End with what surprised you, and which assumption from your Week 2 problem statement was wrong.
    - Keep it in your repo as `interviews/summary.md`. Remove names and anything private.

## DSA: trees and tries

**Focus.** A tree problem is usually one of two shapes. Either each node passes something down (a range, a prefix, a depth), or each node returns something up (a height, a best path, a count). Decide which before you write code. A trie is a tree keyed by characters, so every word that shares a prefix shares a path. That makes "does any word start with this?" cost O(length of prefix), however many words you store.

### Problem list

| Day | Problem | Core idea |
|---|---|---|
| 29 | [98. Validate Binary Search Tree](https://leetcode.com/problems/validate-binary-search-tree/) | Pass a valid range down |
| 29 | [230. Kth Smallest Element in a BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) | In-order walk, stop at k |
| 30 | [105. Construct Binary Tree from Preorder and Inorder Traversal](https://leetcode.com/problems/construct-binary-tree-from-preorder-and-inorder-traversal/) | Root from preorder, split by inorder index |
| 31 | [124. Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) | Return one branch, record both |
| 32 | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | One node per prefix |
| 33 | [211. Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) | Trie plus DFS on wildcards |
| 33 | [212. Word Search II](https://leetcode.com/problems/word-search-ii/) | Walk the grid and the trie together |

### Depth ladder: Validate Binary Search Tree

1. **Solve.** Recurse with `(low, high)` bounds. Each left child tightens `high`, each right child tightens `low`. O(n) time, O(h) stack. Write tests first, including the case where every parent–child pair is valid but the tree is not.
2. **Vary.**
    - Iterative in-order traversal: the values must be strictly increasing. Which version is easier to prove correct?
    - Duplicates are allowed on the left. Which comparison changes?
    - Return the two nodes that were swapped (LeetCode 99, Recover Binary Search Tree).
3. **Break the assumptions.**
    - Use `float("-inf")` and `float("inf")` as starting bounds. Then put a node with value `inf` in the tree. Does your strict comparison reject a valid tree?
    - A tree that is a linked list of 10⁵ nodes. Python's default recursion limit is 1000. What happens, and what is the fix?
    - Node values are `NaN`. Every comparison with `NaN` is `False`. Does your validator accept the tree?
4. **ML lens.** A fitted decision tree is the same structure. Each internal node tests `feature <= threshold`, so the path to a leaf passes down a box of bounds, one range per feature. "Which leaf does this row land in?" is a range walk. Call `decision_path` on a fitted `DecisionTreeClassifier` and print the bounds for one row.
5. **In the wild.**
    - scikit-learn's `Tree` in `sklearn/tree/_tree.pyx` stores the tree as flat arrays: `children_left`, `children_right`, `feature`, `threshold`. Why arrays instead of node objects? What value marks a leaf?
    - SQLite's `src/btree.c` and PostgreSQL's `src/backend/access/nbtree/README`. A database index is a B-tree: many keys per node, so a lookup touches few disk pages. Why does a binary tree make a bad on-disk index?

### Depth ladder: Implement Trie

1. **Solve.** Each node holds a dict of children and an `is_end` flag. `insert`, `search`, and `starts_with` are each O(L) for a word of length L. Write tests for a word that is a prefix of another (`"app"` and `"apple"`).
2. **Vary.**
    - Add `count_prefix(p)`: how many stored words start with `p`? Store a count on each node instead of walking the subtree.
    - Add `delete(word)`. When can you prune nodes, and when must you keep them?
    - Return the top 3 completions for a prefix by stored frequency. Which part is the trie, and which part is a heap (Week 6)?
3. **Break the assumptions.**
    - **Memory.** Insert 10⁵ English words and measure memory with `tracemalloc`. Compare with a plain `set` plus a sorted list and `bisect` for prefix queries. Which wins, and by how much?
    - **Unicode.** `"é"` as one code point or two (Week 1). Does `starts_with("e")` match? Normalize on insert and on query.
    - **Case.** A user types `"Rev"` and you stored `"revenue"`. Decide where case folding happens, and test it.
4. **ML lens.**
    - Tokenizers match the longest known piece at each position. A trie makes that one walk instead of one lookup per candidate length. You will build a tokenizer in Week 7.
    - **Constrained decoding.** When an LLM must output one of a fixed set of labels, a trie of the allowed token sequences says which tokens are legal next. Hugging Face `generate` takes a `prefix_allowed_tokens_fn` for this.
5. **In the wild.**
    - Hugging Face `transformers`: `class Trie` in `src/transformers/tokenization_python.py`. It splits text on added special tokens. Read `split`. How does it handle two added tokens where one is a prefix of the other?
    - Linux IPv4 routing: `net/ipv4/fib_trie.c`. Routing is longest-prefix match on address bits. What does the kernel compress so the trie is not 32 levels deep?

### Depth ladder: Word Search II

- **Vary:** solve it with one DFS per word, then with one DFS over the grid that walks the trie at the same time. Count the cells visited by each on a 12×12 grid with 1,000 words.
- **Break:** all words share a long prefix (`"aaaa...b"`) on a grid full of `"a"`. The search explodes. Prune by removing a word from the trie once found, and remove empty branches. Measure the speedup.
- **ML lens:** walking a trie and a search space together is how constrained decoding prunes beams: stop exploring as soon as no allowed sequence starts with the current prefix.
- **In the wild:** spell checkers and autocomplete. Read how `211`'s wildcard search turns a trie walk into a DFS, then compare it with the trie in `transformers` above, which has no wildcards.

### Shorter ladders: 230, 105, 124

- **Kth Smallest in a BST.** **Vary:** the tree is modified often and you query k often. Store subtree sizes on each node so the query is O(h). **ML lens:** that is an order-statistic tree, the same idea as a streaming quantile for monitoring a feature's median.
- **Construct from Preorder and Inorder.** **Break:** duplicate values make the inorder index ambiguous. Show two different trees with the same two traversals. **ML lens:** serializing a fitted model. scikit-learn's flat arrays are one answer to "what is the minimum you must store to rebuild this tree?"
- **Maximum Path Sum.** **Break:** all values negative. Does your answer start at 0 by mistake? **ML lens:** "return one thing up, record another globally" is the same shape as computing gain at every node while growing a tree.

## ML/DL: tree models, boosting, and leakage

### Learn

- **Decision trees.** Pick the split that most reduces impurity (Gini or entropy for classes, squared error for regression). Trees need no feature scaling and handle mixed feature types.
- **Overfitting in trees.** An unlimited tree memorizes training data. Depth, minimum leaf size, and pruning (`ccp_alpha`) limit it.
- **Random forests.** Average many deep trees trained on bootstrap samples and random feature subsets. Variance goes down.
- **Gradient boosting.** Add shallow trees one at a time. Each new tree fits the gradient of the loss on the current predictions. A small learning rate with more trees usually generalizes better.
- **Histogram boosting.** Bin each feature into at most 255 buckets first, so finding a split is a scan over bins, not over sorted values. This is why it is fast on large data.
- **Leakage.** Information in training that will not exist at prediction time. Three common forms:
    - **Target leakage:** a feature computed from the label, or recorded after the outcome.
    - **Train–test contamination:** preprocessing (scaling, imputing, target encoding) fit on all data before the split.
    - **Split leakage:** random splits on data with time order or repeated entities (the same user in train and test).
- **Feature importance.** Impurity-based importance favors features with many unique values. Permutation importance on held-out data is more honest.

### Read the code

**1. scikit-learn's `GradientBoostingClassifier`** in `sklearn/ensemble/_gb.py`. Answer in your log:

- Find where each stage fits a regression tree to the negative gradient. Why is it a *regression* tree, even for classification?
- Where is `learning_rate` applied? Set it to 1.0 and to 0.01 with the same `n_estimators`. What happens to train and validation loss?
- `subsample < 1.0` turns on stochastic gradient boosting. Which attribute then records out-of-bag improvement?

**2. scikit-learn's `HistGradientBoostingClassifier`** in `sklearn/ensemble/_hist_gradient_boosting/gradient_boosting.py`, and the binning in `binning.py` in the same folder.

- How are bin edges chosen? What happens to a feature with only 3 unique values?
- Where are missing values sent during a split? Find the code that learns a direction for them.
- `early_stopping="auto"` turns on only above a certain number of samples. Find the threshold. Why would a small dataset turn it off?

**3. scikit-learn's `TimeSeriesSplit` and `GroupKFold`** in `sklearn/model_selection/_split.py`.

- What does `gap` do in `TimeSeriesSplit`? When does a gap of 0 leak?
- `GroupKFold` keeps each group in one fold. Write a test that fails if any user ID appears in both train and test.

### Discuss the parameters

| Function | Parameter | Default | What it controls | What goes wrong at the extremes |
|---|---|---|---|---|
| `DecisionTreeClassifier` | `max_depth` | `None` | How deep the tree grows | `None` grows until leaves are pure: perfect train score, poor test score |
| `DecisionTreeClassifier` | `min_samples_leaf` | `1` | Smallest allowed leaf | 1 lets a leaf memorize one row; too large underfits |
| `GradientBoostingClassifier` | `learning_rate` | `0.1` | Weight of each new tree | High overshoots and overfits fast; tiny needs thousands of trees |
| `GradientBoostingClassifier` | `n_estimators` | `100` | Number of boosting stages | Too many overfits unless `n_iter_no_change` stops early |
| `GradientBoostingClassifier` | `max_depth` | `3` | Depth of each stage's tree | 1 means no feature interactions; deep trees make boosting overfit |
| `GradientBoostingClassifier` | `subsample` | `1.0` | Fraction of rows per stage | Below about 0.5, variance rises; 1.0 gives no out-of-bag estimate |
| `HistGradientBoostingClassifier` | `max_bins` | `255` | Bins per feature (255 is the maximum) | Few bins blur thresholds; the cap means fine distinctions are lost |
| `HistGradientBoostingClassifier` | `min_samples_leaf` | `20` | Smallest allowed leaf | Small values memorize rare categories |
| `HistGradientBoostingClassifier` | `early_stopping` | `"auto"` | Hold out a validation split and stop | On time-ordered data the internal split is random, so it leaks the future |
| `TimeSeriesSplit` | `gap` | `0` | Rows skipped between train and test | 0 leaks when features use a trailing window that crosses the boundary |

### Break it

Plant a leak and watch the model find it. Add a column that equals the label plus a little noise, or a timestamp recorded after the outcome. Train, and look at validation accuracy and feature importance. Then write a check that flags any single feature whose AUC alone is above 0.95. Run the check on the clean dataset too. Does it raise false alarms?

## AI agents: memory and context management

### Learn

- **The context window is the only memory the model has.** Everything else is something your code chooses to put back into the prompt.
- **Short-term memory:** the conversation so far. Strategies when it grows too long:
    - Keep the last N turns (sliding window).
    - Summarize older turns into a running summary.
    - Keep the system prompt and pinned facts always, and trim the rest.
- **Long-term memory:** facts saved outside the prompt (a database, files) and retrieved when relevant. The [agents chapter](https://mldsa.github.io/docs/ml-spine/chapter-agents/) covers how memory fits into the agent loop.
- **What to store:** facts the user stated, decisions made, and tool results worth reusing. Not every message.
- **Position matters.** Models use information at the start and end of a long context more reliably than information in the middle ("Lost in the Middle", Liu et al., 2023, [arXiv:2307.03172](https://arxiv.org/abs/2307.03172)).
- **Token budget:** count tokens before every call. Decide in advance what gets cut first.

### Experiment

Write a 40-turn scripted conversation. Plant 10 facts early (a name, a deadline, a preference, a number). At turn 40, ask 10 questions, one per fact. Run it four ways:

- Full history
- Last 10 turns only
- Running summary plus the last 5 turns
- Facts saved to SQLite and retrieved by keyword, plus the last 5 turns

For each, record facts recalled out of 10 and input tokens per call. Plot recall against tokens. Which strategy gives the most recall per token?

### Break it

- Tell the agent a fact, then correct it three turns later ("actually the deadline is Friday"). Which strategies return the old value?
- Ask a summary-based agent about an exact number from turn 3. Summaries round and drop numbers. How often is the number wrong?
- Save a fact from a tool result that contains an instruction ("ignore previous rules"). It now comes back into every prompt. Week 10 covers prompt injection. For now, note where stored memory needs the same checks as user input.

### ML lens

A running summary is lossy compression, like the SVD truncation in the [embeddings chapter](https://dpaul0501.github.io/llm-agent-reads/part_01_embeddings/): you keep what explains most and lose the rest. Retrieval memory is a lookup, and in Week 6 that lookup becomes embedding search.

### In the wild

- LangChain's `trim_messages` in `libs/core/langchain_core/messages/utils.py`. Read the signature. `include_system` defaults to `False`. What happens to your system prompt when you trim with `strategy="last"` and forget to set it?
- MemGPT (Packer et al., 2023, [arXiv:2310.08560](https://arxiv.org/abs/2310.08560)) treats the context window like RAM and external storage like disk, and lets the model call tools to page memory in and out. Which of your four strategies is closest?

## Software dev: SQL and data modeling

### Learn

- Tables, primary keys, foreign keys, `NOT NULL`, `UNIQUE`, `CHECK` constraints
- Normalization: one fact in one place. When to break that rule on purpose.
- Joins, `GROUP BY`, window functions (`ROW_NUMBER`, `LAG`) for time features
- Indexes and `EXPLAIN QUERY PLAN` in SQLite
- Transactions, and parameterized queries (never build SQL with f-strings)
- Schema migrations: a numbered `.sql` file per change, applied in order

### Apply it

Design the schema for this week's build before you write any Python:

- `rows` (the raw data, with an entity ID and an event time), `splits` (which row is in which split, and why), `runs` (one row per training run, with parameters and metrics), `predictions`
- `sessions`, `messages`, and `memories` for the agent, with a foreign key from each memory to the message it came from

Then write the leakage checks as SQL. Two examples: a query that returns entity IDs present in both train and test, and a query that returns test rows whose event time is earlier than the latest train row.

### Verify

- Run `PRAGMA foreign_keys;` on a fresh connection. SQLite does not enforce foreign keys unless you turn them on per connection. Write a test that inserts an orphan memory and expects an error.
- Run `EXPLAIN QUERY PLAN` on the agent's memory lookup. It must use an index, not `SCAN` the whole table. Paste the plan before and after adding the index into your log.
- Write a test that passes a value containing `'; DROP TABLE memories; --` into every query function. The table must still exist.

## Day by day

| Day | DSA | ML/DL | AI agents | Software dev |
|---|---|---|---|---|
| 29 | Validate BST, Kth Smallest | Decision trees; Break it: plant a leak | Context window and token budget | Schema design, keys, constraints |
| 30 | Construct from Preorder and Inorder | **Month 1:** publish blog post | **Month 1:** interview summary and top 3 pains | Load the dataset into SQLite |
| 31 | Max Path Sum | Read `_gb.py`; learning rate experiment | Memory experiment: 4 strategies | Joins, window functions for time features |
| 32 | Implement Trie | Read HistGradientBoosting and binning | Break it: corrections, numbers, stored instructions | Indexes, `EXPLAIN QUERY PLAN` |
| 33 | Add and Search Words, Word Search II | Read `TimeSeriesSplit`, `GroupKFold`; leakage checks | Read `trim_messages`; long-term memory in SQLite | Migrations, injection test |
| 34 | **Weekly build** | | | |
| 35 | **Defend and publish** | | | |

## Weekly build

### Leak-checked model with a memory agent

```bash
leakcheck load data.csv --entity user_id --time event_time   # into SQLite
leakcheck train --split time                                 # train, check, record the run
leakcheck chat --session alice                               # agent with memory
```

Use a tabular dataset from your problem area if you have one. If not, use a public dataset with a time column and repeated entities, so that leakage is possible.

**Required behavior**

- **Data (Software dev).**
    - Load the CSV into SQLite using your schema, with foreign keys on.
    - Every split, run, and prediction is a row. Nothing important lives only in memory.
- **Model (ML/DL).**
    - A baseline (majority class or logistic regression), then a gradient boosting model.
    - All preprocessing inside a scikit-learn `Pipeline`, fit on train only.
    - A time-based or group-based split, chosen and justified in the README.
- **Leakage checks (ML/DL + Software dev).**
    - No entity in both train and test. No test row earlier than the latest train row (for time splits).
    - No duplicate rows across splits (row hashes, as in Week 1).
    - Flag any single feature whose AUC alone is above 0.95.
    - Training refuses to record a run if a check fails.
- **Agent (AI agents).**
    - Your Week 4 agent loop, with tools to query runs, explain a prediction, and save a memory.
    - Memories persist in SQLite across sessions. Start a new session and it still knows what you told it.
    - A token budget per call. The system prompt and pinned memories are never trimmed.
- **Name lookup (DSA).**
    - A trie of column names and entity IDs. The agent's tool arguments are checked against it. Unknown names are rejected with the three closest prefix matches.

**Stretch**

- Compare impurity-based and permutation importance on the test set. Show a feature they disagree on, and explain why.
- Add a running summary to the agent and rerun the Day 31 memory experiment against your own build.

## Publish

**Every day:** one LinkedIn post, **Day N/100**, in the [standard format](../how-it-works.md#the-daily-linkedin-post). Suggested findings, one per day:

| Day | Post idea |
|---|---|
| 29 | "A decision tree is a BST of ranges." Print the bounds `decision_path` gives for one row. |
| 30 | Your Month 1 blog post, with the one measurement it is built on. |
| 31 | "Learning rate 1.0 vs 0.01, same 100 trees." Show the train and validation loss curves. |
| 32 | "I stored 100k words in a trie and in a set." Show the memory numbers. |
| 33 | "I planted a leak. The model found it in one split." Show the feature importance and the check that catches it. |
| 34 | Your build: a new session where the agent recalls a fact from yesterday, with the SQL row it came from. |
| 35 | What the defense exposed, and what you fixed. |

**This week's write-up:** the **Month 1 technical blog post** on Medium, published on Day 30. See [Month 1 milestone](#month-1-milestone).

## Value-creation step

Turn your 5 interviews into the **top 3 pains**, ranked by how many people raised them and how much each one costs them today. This is the second half of the Day 30 milestone. Pick the one pain you will build for. In [Week 6](week_06.md) you will sketch an MVP for it and show it to two of the people you interviewed.
