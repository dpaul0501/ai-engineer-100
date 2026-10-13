# How Each Week Works

Every week opens with an **anchor page** that states what you will learn, the goal for the week, and the challenge you must build and defend. The structure is the same every week, so you always know where to look.

## The weekly rhythm

The program is **self-paced**. Day N is a step, not a date: start when you are ready, and move on when you finish a day. The daily tasks are in the private daily guide you get after you [sign up](https://forms.gle/RSezwZ7xUM5WRHiE8).

| When you reach | What happens |
|---|---|
| The mock interview day of a week | Pair with a peer at a similar stage; a moderator observes |
| The end of a week | Book your defense with a moderator |
| Day 30, 60, or 90 | Present at the next monthly idea review |
| Day 100 | Present at the next demo day |

**Months 1 and 2 (Days 1–60).** Most weeks have seven days:

| Day of the week | What happens |
|---|---|
| 1–5 | Study all four dimensions. Keep a daily learning log in your repo. Post on LinkedIn every day. |
| 5 | **Peer mock interview:** 45 minutes in pairs, with a moderator observing. |
| 6 | **Build day:** write a design doc first, then build one working piece that uses all four dimensions. |
| 7 | **Defend:** a 15-minute live review. Then publish a longer write-up of your best finding. |

Weeks 4 and 8 have nine days. They end with the month project, the defense, and the idea review on Day 30 and Day 60.

**Month 3 (Days 61–100).** About 30% of each day is study (one harder DSA problem and one advanced topic). About 70% is the big build for your primary path. Peer mock interviews continue every week, and each week ends with a **sprint review** of your build and your path evidence.

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

At the end of each week you demo the weekly build to a moderator and your peer group, and they probe it:

- The moderator or a peer picks any line. You explain it and say what breaks if it changes.
- You show the test that would catch that break. If no test catches it, that is a finding to fix.
- You state which parts AI wrote and how you verified each one.

## The peer group

This is a peer group, not a class. You learn as much from reviewing other people's work as from doing your own.

| Who | What they do |
|---|---|
| **Peers** | Interview each other, review each other's pull requests and posts, and question each other's builds at the end-of-week defense |
| **Moderators** | Run the end-of-week defenses and sprint reviews, observe and score the peer mock interviews, and keep the standard high |
| **Technology leaders** | Industry practitioners who give feedback at the idea reviews and demo day, and run industry mock interviews when they are available |

### Peer mock interviews (every week)

Pair with another participant. One interviews and one solves, then you swap. A moderator observes and scores.

#### Mock interview format

| Minutes | What the candidate does |
|---|---|
| 0–5 | Restate the problem. Ask about input size, edge cases, and what to return. Write two examples. |
| 5–10 | Say the brute force and its complexity. Then name the pattern and the better approach. Get a nod before coding. |
| 10–35 | Code while thinking aloud. Say what each block does and why. |
| 35–45 | Test by hand: the examples, then an edge case you choose. State the final time and space complexity. |

- Pick one problem from the current week's list that your partner has not solved yet.
- No AI, no autocomplete, no searching.
- The interviewer scores clarifying questions, choice of approach, correctness, and testing, 1–4 each. The moderator checks the scores and adds one note for each of you.
- Rotate partners every week, so you interview with everyone in the group.

### Industry mock interviews

Technology leaders run mock interviews as their availability allows, in the same format as the peer interviews. They are not scheduled weekly. When a slot opens, it goes to participants whose recent peer mock scores show they are ready.

### Idea reviews (Days 30, 60, and 90)

Present your progress on both paths to a panel of technology leaders and moderators:

- **5 minutes:** the problem or question, what you built, and the evidence it mattered to someone
- **10 minutes:** questions and feedback from the panel
- Leave with one thing to stop, one thing to start, and one person to contact

Demo day on Day 100 uses the same panel.

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
