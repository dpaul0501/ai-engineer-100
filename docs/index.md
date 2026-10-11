# 100 Days: Zero to One AI Engineer

The goal is not to finish a syllabus. The goal is to become someone who creates value: learn a fundamental in depth, build with it, prove it works, explain it in public, and put it in front of people who will use it or pay for it.

[Open the orientation slides](orientation.html){ .md-button .md-button--primary } [How each week works](how-it-works.md){ .md-button }

## What you walk away with

Proof, not a certificate.

=== "Students"

    - 100 interview problems, each taken past the solution to variations, failure cases, and real code that uses the idea
    - 14 builds on your public GitHub, each one defended live
    - 100 public posts: a track record recruiters can read
    - A product with real users, and a paper or patent attempt

=== "Working professionals"

    - The internals behind the AI tools you already use: attention, training, inference, RAG, and agents
    - A disciplined way to work with coding agents: spec, review, eval, reject
    - A public portfolio for moving into AI and ML roles
    - A side product with users, and a first paying customer or letter of intent

## From preparation to creating value

| Stage | Weeks | What happens | Proof |
|---|---|---|---|
| **Prepare** | 1–3 | Fundamentals in four tracks, and the habit of proving your code works | Tested tools, gradient checks, accuracy with an error bar |
| **Point it at a real problem** | 4–7 | Your own data, your own problem area, five interviews with people who have the problem | Day 30 article, interview notes, RAG with evals in CI |
| **Ship to users** | 8–10 | Model internals from scratch while an MVP reaches ten real people | Usage logs from 10 users, a technical report |
| **Create value** | 11–14 | Production, a first paying user, a paper or patent, and a public launch | Revenue or a letter of intent, demo day |

### Believe it, then test it

Believe you can build something people want in 100 days. Then test that belief every week:

- **Code:** tests that fail when the code is wrong
- **Models:** evals with an error bar, not one lucky run
- **Understanding:** a live defense, line by line
- **Ideas:** interviews about what people already do, not what they say they would do
- **Product:** usage logs from real people
- **Value:** someone pays, or signs a letter of intent

Test the program the same way. Do [Week 1](weeks/week_01.md) Day 1 today. If the depth ladder has not changed how you read code by Day 7, this program is not for you.

## Four tracks, running in parallel

Every day moves each track forward by one step. By Day 100 each track has 100 steps behind it.

| Track | Daily dose | Day 1 | Day 100 |
|---|---|---|---|
| **DSA** | 1 problem, taken through the depth ladder (~45m) | Arrays and hashing | Graphs, dynamic programming, mock interviews |
| **ML/DL** | 1 concept + 1 code read (~60m) | NumPy and vectors | Training and serving a small GPT |
| **AI agents** | 1 concept + 1 experiment (~60m) | A single LLM call | A production agent inside your product |
| **Software dev** | 1 engineering practice (~45m) | git and pytest | CI/CD, security, deployment |

DSA uses a curated list of about 100 problems grouped by pattern, not LeetCode #1–100 in order.

## Every day, every week, every month

| Rhythm | What you do |
|---|---|
| **Every day** | Move all four tracks forward. Post one finding on LinkedIn as **Day N/100**. |
| **Every week** | Build one working piece that uses all four tracks, defend it live, and publish a longer write-up. |
| **Every month** | Communicate at a higher level and take a step toward real users. |

### Monthly milestones

| Milestone | Communication | Users and value |
|---|---|---|
| Day 30 | Technical blog post | 5 interviews with people you know about the problem you are solving |
| Day 60 | Technical report or paper reproduction (arXiv-style) | MVP used by 10 people |
| Day 90 | Workshop paper submission, or a provisional patent | First paying user, or a signed letter of intent |
| Day 100 | Demo day and public launch | Users, revenue, and lessons presented |

## Week by week

What each week covers (DSA · ML/DL · AI agents · software dev), and what you can prove by the end of it. Each weekly build is a feature of your own product, so the capstone exists by Week 14.

| Wk | Days | Focus | Coverage | By Sunday you can |
|---|---|---|---|---|
| [1](weeks/week_01.md) | 1–7 | [Lookups, Vectors, and Your First LLM Call](weeks/week_01.md) | Hashing · NumPy, vectors · LLM API calls · git, pytest | Build a tested search tool from scratch and defend every line |
| [2](weeks/week_02.md) | 8–14 | [Pointers, Matrices, and Structured Output](weeks/week_02.md) | Two pointers · linear algebra, probability · structured output · TDD | State your LLM tool's accuracy with an error bar |
| [3](weeks/week_03.md) | 15–21 | [Stacks, Gradients, and Tool Calls](weeks/week_03.md) | Stack, binary search · autograd · tool calling · debugging | Prove your autograd engine's gradients are right, and let an LLM call it |
| [4](weeks/week_04.md) | 22–28 | [Lists, Trees, Models, and the Agent Loop](weeks/week_04.md) | Lists, trees · regression, metrics · agent loop · REST APIs | Serve a model on your own data, called by an agent you wrote without a framework |
| [5](weeks/week_05.md) | 29–35 | [Trees, Leakage, and Memory](weeks/week_05.md) | Tries · boosting, leakage · memory · SQL | Prove your model does not leak. **Day 30:** article + 5 interviews |
| [6](weeks/week_06.md) | 36–42 | [Heaps, Neural Nets, and Retrieval](weeks/week_06.md) | Heaps · PyTorch, backprop · RAG · Docker, CI | Show with numbers how often retrieval finds the right chunk |
| [7](weeks/week_07.md) | 43–49 | [Graphs, Tokens, and Judges](weeks/week_07.md) | BFS/DFS · tokenization, embeddings · LLM-as-judge · PR review | Explain every merge your tokenizer made, and measure your judge's agreement |
| [8](weeks/week_08.md) | 50–56 | [Search Trees, Attention, and Agents That Plan](weeks/week_08.md) | Backtracking · RoPE, attention · multi-agent · system design | Write multi-head attention from a blank file and match PyTorch |
| [9](weeks/week_09.md) | 57–63 | [Recurrences, the Transformer Block, and Tools Over a Protocol](weeks/week_09.md) | 1-D DP · transformer block, training · MCP · observability | Reproduce a published claim. **Day 60:** report + 10 users |
| [10](weeks/week_10.md) | 64–70 | [Alignment Tables, Generation, and Defending Your Agent](weeks/week_10.md) | 2-D DP · KV cache, sampling · guardrails · security | Sample from your own GPT, and report your agent's attack success rate |
| [11](weeks/week_11.md) | 71–77 | [Intervals, Fine-Tuning, and Coding Agents](weeks/week_11.md) | Intervals, greedy · LoRA · coding agents · profiling | Show whether fine-tuning beats the prompt-only baseline |
| [12](weeks/week_12.md) | 78–84 | [Weighted Graphs, Honest Evaluation, and Production](weeks/week_12.md) | Weighted graphs · experiment design · cost, latency · deploy | Run for real users, with cost and p95 latency per task |
| [13](weeks/week_13.md) | 85–91 | [Bits, Research, and the Month 3 Milestone](weeks/week_13.md) | Bits · research week · capstone agent · open source | **Day 90:** paper or patent, and a first paying user or letter of intent |
| [14](weeks/week_14.md) | 92–100 | [Mock Interviews, Polish, and Demo Day](weeks/week_14.md) | Mock interviews · capstone polish · launch checklist | **Day 100:** demo live, launch, present users and revenue |

## What you need

- A laptop that runs Python 3.11 or newer
- A GitHub account and a LinkedIn profile
- About 4 hours on weekdays, plus the weekend build
- An API key for one LLM provider (from Week 1)
