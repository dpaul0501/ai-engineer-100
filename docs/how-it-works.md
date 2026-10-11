# How Each Week Works

Every week opens with an **anchor page** that states what you will learn, the goal for the week, and the challenge you must build and defend. The structure is the same every week, so you always know where to look.

## The weekly rhythm

| Days | What happens |
|---|---|
| Mon–Fri | Learn across all four tracks. Keep a daily learning log in your repo. Post on LinkedIn every day. |
| Sat | **Weekly build:** one working piece that uses all four tracks. |
| Sun | **Defend:** a 15-minute live review. Then publish a longer write-up of your best finding. |

## The depth ladder

Solving a problem is the first rung, not the last. Every DSA problem goes through five rungs:

1. **Solve.** Write it yourself, with tests. State time and space complexity.
2. **Vary.** Change the problem: sorted input, a stream, k instead of 2, floats instead of ints. What changes in the solution?
3. **Break the assumptions.** What does your code assume? Duplicates, empty input, overflow, Unicode, data larger than memory, adversarial input. Show a failing case, then fix it or document the limit.
4. **ML lens.** Where does this idea live inside an ML system? Find the place and explain the connection.
5. **In the wild.** Find real software that uses it. Read the actual source file and write down what it does differently from your version, and why.

ML/DL concepts follow the same idea with three steps:

- **Learn** the concept.
- **Read the code** of a real implementation.
- **Discuss the parameters:** list every argument, and explain what it controls, what its default is, and what goes wrong at the extremes.

## The defense

Each Sunday you demo the weekly build and a reviewer probes it:

- The reviewer picks any line. You explain it and say what breaks if it changes.
- You show the test that would catch that break. If no test catches it, that is a finding to fix.
- You state which parts AI wrote and how you verified each one.

## AI use, by phase

| Days | Rule |
|---|---|
| 1–20 | **AI explains, you type.** AI may explain and review. It does not write your code. |
| 21–60 | **AI pairs, you prove.** AI may write code. Tests come first, and you must explain every line it wrote. |
| 61–100 | **AI agents, you review.** Agents write code. You write the spec, review the PR, run the evals, and reject bad output. |

## The daily LinkedIn post

Every day ends with one post. Same format every day, so the series is recognizable:

```text
Day 23/100 · #100DaysAIEngineer

<The problem or concept, in one line>
<What broke when I changed an assumption, or what the parameter really does>
<Where real software does this, with a link to the source file>

Code: <link to your repo>
```

**The quality bar:** one finding per post, with evidence: code, a benchmark, or a link to a real source file. "Today I learned X" is not a finding.

**The community rule:** comment usefully on at least two other participants' posts every day. A useful comment adds a test case, a counterexample, or a link, not "great post".

## The weekly write-up

Once a week, turn the best finding into something longer. The channel rotates:

| Channel | Signal that counts |
|---|---|
| GitHub | A README write-up that a stranger can reproduce |
| Kaggle | A notebook, ideally with a public score |
| Medium | An article with code and real source links |
| Hackathon | A submitted entry |

Stars and upvotes count only when paired with a stronger signal: a real user, a reviewer's sign-off, or a placement.

!!! warning "Publishing and patents"
    In India and most other countries a patent requires absolute novelty. A public post or repo that describes your invention can make it unpatentable. If you plan to file in Month 3, file a provisional application **before** publishing anything about that idea. Keep posting about the other parts of your work.
