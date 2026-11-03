# 100 Days: Zero to One AI Engineer

The goal is not to finish a syllabus. The goal is to become someone who creates value: learn a fundamental in depth, build with it, prove it works, explain it in public, and put it in front of people who will use it or pay for it.

**Start preparing from Day 1 of your studies.** You do not need to wait for a degree to start building the profile that makes companies, clients, and collaborators want to work with you.

This is a **peer group**, not a class. Participants review each other's work and run mock interviews with each other. Moderators keep the standard, and technology leaders from industry give feedback on your ideas.

[Start the guide](https://dpaul0501.github.io/ai-engineer-100-labs/){ .md-button .md-button--primary } [Sign up](https://forms.gle/RSezwZ7xUM5WRHiE8){ .md-button } [Orientation slides](orientation.html){ .md-button } [What you build](what-you-build.md){ .md-button }

**Self-paced.** Start whenever you are ready. Day N is a step, not a date. Peers are matched by the stage they have reached, not by start date.

## What you walk away with

Proof, not a certificate.

=== "Students"

    - 100 interview problems, each taken past the solution to variations, failure cases, and real code that uses the idea
    - Weekly builds and two month projects on your public GitHub, each one defended live, grown into one big build in Month 3
    - 100 public posts: a track record recruiters can read
    - Evidence from the two paths you choose: merged open-source work, a paper, hackathon placements, paying clients, or a product with users

=== "Working professionals"

    - The internals behind the AI tools you already use: attention, training, inference, RAG, and agents
    - A disciplined way to work with coding agents: spec, review, eval, reject
    - A public portfolio for moving into AI and ML roles
    - Evidence from the two paths you choose, for example open-source contributions, a paper, or paying consulting clients

## From preparation to creating value

| Month | Days | Focus | Study / build | Month project |
|---|---|---|---|---|
| **1: AI Engineer** | 1–30 | Tokenization, prompts, context and memory, RAG and search, agents | ~80% / 20% | An agentic harness built from your own parts |
| **2: Deep learning** | 31–60 | Neural nets and autograd, attention and a GPT, vision and multimodal, generative models | ~80% / 20% | A generative model, released with a model card |
| **3: Core ML engineering + validation** | 61–100 | Optimization, training at scale, inference, post-training, and one big build aimed at your chosen path | **30% / 70%** | The Month 1 and 2 projects grown into something bigger: a merged feature, a paper or patent, hackathon wins, a paying client, or a product with users |

### Believe it, then test it

Believe you can build something people want in 100 days. Then test that belief every week:

- **Code:** tests that fail when the code is wrong
- **Models:** evals with an error bar, not one lucky run
- **Understanding:** a live defense, line by line
- **Ideas:** interviews about what people already do, not what they say they would do
- **Product:** usage logs from real people
- **Value:** a maintainer merges it, a reviewer accepts it, judges rank it, or someone pays for it

Test the program the same way. Do Week 1 Day 1 today. If the depth ladder has not changed how you read code by Day 7, this program is not for you.

## Five dimensions

Every week moves each dimension forward, inside one build. In Month 3, study narrows to advanced topics and the build takes most of the day.

| Dimension | Daily dose | What it covers |
|---|---|---|
| **DSA** | 1–2 problems through the depth ladder (~45m) | Every LeetCode pattern, each placed in the week whose build uses it |
| **ML** | 1 concept, derived and checked numerically (~45m) | Statistics and core ML: probability, linear algebra, calculus, optimization, measurement |
| **AI** | The week's build (~2h) | Tokenization, prompts, context and memory, retrieval, agents, then the models themselves |
| **System Design** | 2 design reps a week | The Google and Meta product, infrastructure, ML and LLM rounds, with numbers |
| **Software Delivery** | Every build | Spec → contract → test → code, property tests, planted bugs, CI, release |

DSA problems are grouped by pattern and tied to the build, not LeetCode #1–100 in order.

## Every day, every week, every month

| Rhythm | What you do |
|---|---|
| **Every day** | Move all five dimensions forward. Post one finding on LinkedIn as **Day N/100**. |
| **Every week** | Build one working piece that uses all five dimensions, defend it live, run a peer mock interview, and publish a longer write-up. |
| **Every month** | Hit the next milestone on each of your two paths, and present it to technology leaders at the idea review. |

### Pick two paths

Beyond the core, every participant picks two of five paths to create value, one primary and one secondary. Each path has its own Day 30, 60, 90, and 100 milestones.

| Path | You prove that |
|---|---|
| **Open source** | You can ship code that strangers review and merge |
| **Academic papers** | You can find a real question and answer it with evidence |
| **Hackathons** | You can build something that works, under a deadline |
| **Consultancy** | Someone will pay you to solve their problem |
| **Product** | Many people want a thing you built |

See [Choose your paths](paths.md) for milestones and suggested combinations.

## Week by week

The weekly pages open in order. Month 1 is below; later weeks appear in the guide as they are released.

| Wk | Days | Build | DSA | ML | AI | System Design |
|---|---|---|---|---|---|---|
| 1 | 1–7 | **[Tokenization](https://dpaul0501.github.io/ai-engineer-100-labs/week_01/):** byte-level BPE from first principles, with a tested tokenizer of your own | Hashing, sliding windows, greedy vs optimal, Huffman, heaps with lazy deletion, linked lists | Zipf's law, the length / vocabulary trade-off, entropy and bits per byte | The token tax, split patterns, glitch tokens | Training on terabytes · serving a tokenizer |
| 2 | 8–14 | **Prompt, context and memory engineering** | Sliding window, LRU/LFU caches, intervals | Entropy, similarity | Context assembly, compaction, short- and long-term memory | ◆ Rate limiter · Distributed cache |
| 3 | 15–21 | **RAG and search:** inverted index, BM25, HNSW, hybrid retrieval, GraphRAG | Graphs, heaps, k-way merge, tries | TF-IDF/BM25, embeddings, nDCG | Chunking, reranking, GraphRAG | ◆ Web search · Autocomplete |
| 4 | 22–30 | **Month 1 project: the agentic harness:** agent loop, tools, skills, MCP | State machines, backtracking, DAGs | pass@k, bootstrap intervals | Tools, skills, MCP, subagents, permissions | ◆ Chat · Job scheduler |

Every week also has practice projects on real data (a Kaggle competition when one fits the week's statistics), a capstone design, and a live defense. Months 2 and 3 cover deep learning (neural nets, attention, vision, diffusion) and core ML engineering (optimization, training at scale, inference, post-training).

## What you need

- A laptop that runs Python 3.11 or newer
- A GitHub account and a LinkedIn profile
- About 5 hours a day, including the build

## Join

The [guide](https://dpaul0501.github.io/ai-engineer-100-labs/) is free and open to anyone: one page per week, with the tasks, labs, and starter code. Start whenever you are ready.

To join the peer group (mock interviews, defenses, idea reviews, and demo day), [sign up here](https://forms.gle/RSezwZ7xUM5WRHiE8).
