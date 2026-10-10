# Task Scheduler: Solving LeetCode 621 With the Heap Pattern Instead of Memorizing the Formula

"Task scheduler" lands people in three completely different places. One group wants Windows Task Scheduler — the GUI with Triggers and Actions. Another group means cron, systemd timers, or a delayed-job queue in a backend service. The third group is sitting in front of LeetCode 621 and wondering why the answer is `max((maxFreq - 1) * (n + 1) + countOfMaxFreq, len(tasks))`.

This article is for the third group, with a short detour for the second. If you want the Windows tool, you're in the wrong tab.

## Which "task scheduler" you actually need

Quick way to tell:

- You're naming triggers, actions, and "run whether user is logged on or not" — that's Windows Task Scheduler, and the interview questions about it are about the GUI and service accounts, not algorithms.
- You're writing `crontab -e`, `systemd` timers, Celery beat, or a `ScheduledExecutorService` — that's real-job scheduling, and the hard problems are persistence, retries, time zones, and clock drift.
- Someone handed you an array of characters and a number `n` and asked for the "minimum number of CPU intervals" — that's LeetCode 621, and it's the one that shows up in coding screens.

The third case is where people actually get stuck, because the problem looks like simulation but the clean answer is arithmetic. Both approaches matter, and knowing when to reach for which is the real skill.

## What the Task Scheduler problem actually asks

You get an array of tasks labeled `'A'` to `'Z'`. Each task takes one unit of time. Between two executions of the same task there must be at least `n` intervals — which can be filled by other tasks or by idle time. Return the minimum number of intervals needed to finish everything.

Constraints: up to 10⁴ tasks, `0 <= n <= 100`.

Three worked examples worth internalizing:

| Input | n | Answer | One valid schedule |
| --- | --- | --- | --- |
| `["A","A","A","B","B","B"]` | 2 | 8 | A → B → idle → A → B → idle → A → B |
| `["A","C","A","B","D","B"]` | 1 | 6 | A → B → C → D → A → B |
| `["A","A","A","B","B","B"]` | 3 | 10 | A → B → idle → idle → A → B → idle → idle → A → B |

Notice the third row: with only two task types and a cooldown of 3, you have no choice but to idle twice per round. That's the whole problem in miniature. The answer isn't "just run the tasks back to back" — sometimes the CPU genuinely has nothing legal to do.

## The observation that unlocks it

The most frequent task is the bottleneck. Everything else fits around it.

If task A appears `maxFreq` times and the cooldown is `n`, then A alone forces a structure of `maxFreq - 1` blocks, each of width `n + 1`: one execution of A plus `n` slots that must be filled with something else (another task, or idle).

For `["A","A","A","B","B","B"]` with `n = 2`, that frame looks like:


A B _        <- block 1
A B _        <- block 2
A B          <- final occurrence, no trailing cooldown needed


Two full blocks of 3, plus a tail containing every task that ties for the maximum frequency. That's `(3 - 1) * 3 + 2 = 8`.

Two details people miss:

1. The tail isn't "one task." It's every distinct task whose count equals `maxFreq`. Two tasks tied at 3 means the tail is width 2.
2. If you have enough distinct tasks to fill every gap, no idle time is needed at all and the answer collapses to `len(tasks)`. That's the case for `["A","A","A","A","B","B","B","C","C"]` with `n = 1`: the frame formula predicts 7, but you need 9 intervals because you have to run all 9 tasks. Hence the `max()`.

## Approach 1: simulate with a max-heap and a cooldown queue

This is the version most people reach for first, and it's the one that's easier to explain out loud.

The idea: at every interval, run the task with the highest remaining count, because the most frequent task is the one that's hardest to schedule. After running it, park it in a cooldown queue with the time it becomes legal again.

python
import heapq
from collections import Counter, deque

def least_interval(tasks, n):
    counts = Counter(tasks)
    heap = [-c for c in counts.values()]
    heapq.heapify(heap)

    cooldown = deque()   # (ready_time, remaining_count)
    time = 0

    while heap or cooldown:
        time += 1

        if cooldown and cooldown[0][0] == time:
            _, remaining = cooldown.popleft()
            heapq.heappush(heap, -remaining)

        if heap:
            remaining = -heapq.heappop(heap) - 1
            if remaining > 0:
                cooldown.append((time + n + 1, remaining))
        # else: heap is empty and nothing is ready — this interval is idle

    return time


Trace for `["A","A","A","B","B","B"]`, `n = 2`:


t=1  run A     (A: 3→2, ready again at t=4)
t=2  run B     (B: 3→2, ready again at t=5)
t=3  heap empty, nothing ready → idle
t=4  A returns, run A     (A: 2→1, ready at t=7)
t=5  B returns, run B     (B: 2→1, ready at t=8)
t=6  idle
t=7  run A     (A: 1→0, done)
t=8  run B     (B: 1→0, done)


Complexity: there are at most 26 distinct labels, so every heap operation is `O(log 26)` — effectively constant. Total time is `O(N)` in the number of intervals, extra space `O(1)` (26 counters plus a cooldown queue that holds at most `n + 1` entries).

Two implementation traps:

- Don't store the task characters in the heap. You only care about counts. Storing letters invites you to build a heap of size `N` and turn an `O(N)` solution into `O(N log N)` for no benefit.
- The cooldown queue entry has to record the *next legal time*, not "how many intervals left." Counting down is where off-by-one bugs live.

## Approach 2: the greedy/arithmetic formula

Once you've written the simulation, the formula is a short step away. Count the frequencies, take `maxFreq` and the number of tasks tied at that frequency, then:

python
from collections import Counter

def least_interval(tasks, n):
    counts = Counter(tasks)
    max_freq = max(counts.values())
    count_of_max = sum(1 for c in counts.values() if c == max_freq)
    return max((max_freq - 1) * (n + 1) + count_of_max, len(tasks))


That's `O(N)` time for the counting pass and `O(1)` space. No heap, no queue, no simulation loop.

Why the `max()` with `len(tasks)` matters: when there are many distinct task types, the frame's gaps all get filled with real work, so the answer is simply "run everything back to back." The frame formula underestimates in that case, which is exactly the bug that catches people who memorize the arithmetic without understanding it.

Sanity checks worth running mentally:

- `n = 0`: formula gives `(maxFreq - 1) * 1 + count_of_max`, which equals `len(tasks)`. Correct — no cooldown means no idle.
- One distinct task, three copies, `n = 2`: `(3 - 1) * 3 + 1 = 7`. Correct — A, idle, idle, A, idle, idle, A.
- All distinct tasks: `maxFreq = 1`, so the frame term is `0 * (n + 1) + k`, and `len(tasks)` wins. Correct — no idling.

## Which one should you present first?

Start with the simulation, land on the formula. That order signals that you understand the scheduling mechanics, not just a closed-form trick.

More practically: the formula tells you the *length* of the optimal schedule, not the schedule itself. If the interviewer asks you to output the actual ordering, the frame formula is useless and the heap version is already finished. Same thing if the problem changes shape.

Also worth saying out loud in an interview — and this is the kind of remark that separates a hire from a near-miss — the formula generalizes poorly. Heap-plus-queue generalizes well. If you're preparing for coding screens rather than one specific question, that distinction is the actual takeaway.

## Five follow-ups that reuse the same spine

This is the reason to learn the pattern instead of the problem. Heap + deferred/re-entry structure shows up across a whole family:

| Problem | What changes in the spine |
| --- | --- |
| 767. Reorganize String | Cooldown collapses to `n = 1`, so the queue becomes a single "previous character" slot. Add a feasibility check first: if the most frequent character exceeds `ceil(len / 2)`, no arrangement exists |
| 358. Rearrange String k Distance Apart | Cooldown becomes a parameter again, but you must return the rearranged string, and detect impossibility |
| 1834. Single-Threaded CPU | Priority key flips from frequency to `(duration, arrival_index)`. The cooldown queue becomes an arrival-time event queue that gates what's eligible |
| 1942. Smallest Unoccupied Chair | The single stack forks into two heaps — one for free chairs, one for `(leave_time, chair)` pairs — that hand resources back and forth |
| "Design a task scheduler" (system design) | Same conceptual core, completely different constraints: durable storage, retries with backoff, at-least-once execution, clock skew, and what happens when the scheduler process dies mid-job |

One concrete bug worth knowing before you hit 1834: the cumulative time can reach 10¹⁴ (10⁵ jobs × 10⁹ processing time). Java needs `long`, C++ needs `long long`, Go needs `int64`. Python's integers grow on demand, which is why the same bug never shows up in Python solutions and then mysteriously appears when you rewrite in Java.

## Why this pattern is hard to see on your own

Ask anyone who's failed a coding screen and the answer is rarely "I couldn't write the code." It's "I spent fifteen minutes trying approaches that didn't fit." Recognizing that a problem is a *heap with a holding pen* problem is a different skill from implementing a heap.

Platforms built around problem volume don't teach that skill directly — they let you infer it after your fifth or sixth similar problem. Pattern-first platforms teach it explicitly, which is the entire pitch behind AlgoMonster.

That's the honest framing of what the platform is: a text-and-diagram course built by ex-Google engineers, organized around named patterns rather than a problem list. The Pro subscription unlocks all patterns, lessons, and problems, unlimited AI assistant usage, unlimited Speedrun (a quick-review mode for cycling through problems you've already seen), and company-tagged questions that get refreshed monthly. The site also advertises 500+ lessons across 6 courses, 1,000+ illustrations, and 3,000+ editorials, and throws in bonus courses like CS fundamentals and a "build an LLM from scratch" course.

Where it's weak — and this matters if you're deciding where your money goes:

- There's no video. Everything is text and interactive diagrams. If you learn by watching someone code, this format will frustrate you.
- The problem bank is small by design. Reviewers put it around 231 mapped lessons and problems, versus LeetCode's 3,000+. You'll likely still need LeetCode for raw volume before a Google loop.
- System design coverage is introductory. It won't carry you through an L5+ design round.
- No live mocks, no coaching, no human feedback on your solutions.

Several third-party reviews also mention a job-offer guarantee attached to completing the curriculum — worth reading the terms on the official site yourself before treating it as a factor.

## Every plan AlgoMonster currently lists

The pricing page has changed shape more than once, so here's what the official subscribe page showed during this research:

| Plan | What you get | Price | Billing | Get it |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and problems — enough to judge the teaching style | $0 | — | [Start free on AlgoMonster](https://algo.monster/subscribe?fpr=li69) |
| Pro – Monthly | All patterns, lessons, and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; 1 month of full access | $18.99/mo | Monthly, cancel anytime | [Get AlgoMonster Pro Monthly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Yearly | Everything above plus 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials, member discount on live bootcamps; 12 months of full access | $5.84/mo, billed annually ($69.99/year) | Annual | [Get AlgoMonster Pro Yearly](https://algo.monster/subscribe?fpr=li69) |
| Pro – Lifetime | Everything in the annual plan plus lifetime updates | $89 | One-time | [Get AlgoMonster Pro Lifetime](https://algo.monster/subscribe?fpr=li69) |

Two caveats, stated plainly. First, AlgoMonster runs promotions often enough that the number you see can differ from the number above — check the live page before you decide. Second, other reviews quote figures like $99/year or $189 lifetime, and one quotes $459 lifetime. Those snapshots don't match each other or the page I checked, so treat any price you read in a blog post (including this one) as a starting point and confirm at checkout.

The math that actually matters: if the monthly tier is still available at $18.99, it costs less than the yearly plan if you finish the curriculum in three months instead of twelve. AlgoMonster's own material describes the curriculum as designed for busy people and completable in weeks, not a year — so a monthly plan for a single job search, and lifetime if you expect to interview again in a couple of years, is the more sensible split.

## Who actually benefits

You'll get your money's worth if you already write working code and your failure mode is *choosing* the wrong approach — if Task Scheduler took you forty minutes not because the heap was hard but because you didn't see that it was a heap problem.

You'll waste it if you're still shaky on fundamentals (the platform assumes fluency and teaches selection, not syntax), if you need video, or if your interview loop is system-design heavy. In that last case, budget separately.

If you want to test the format before paying anything, the free tier exists precisely for that. 👉 [Try the free AlgoMonster tier first](https://algo.monster?fpr=li69) and work through whichever pattern chapter covers heaps and priority queues.

## Quick answers

**Can I solve Task Scheduler with sorting only?** Yes — count frequencies, sort them descending, then compute idle slots as `(maxFreq - 1) * n` minus the contributions of the other tasks, capped at zero, and add `len(tasks)`. It's the same logic as the formula with an extra sorting step (`O(N log N)` instead of `O(N)`).

**Is the heap version ever wrong?** Not if the cooldown bookkeeping is right. The two common failures are forgetting to push a task back when its count is still positive, and comparing against the wrong time offset (`time + n` instead of `time + n + 1`, or the reverse).

**Why does the answer sometimes equal `len(tasks)`?** Because the gaps are all filled with real work. Idle time only appears when the most frequent task forces a gap wider than the supply of other tasks.

**Does the pattern transfer past coding interviews?** The system-design version does, and it's harder in different ways: durable queues, retries, idempotency, and what happens when the scheduler dies holding a job. Heap-with-a-holding-pen gets you through the coding screen; it won't get you through the design round on its own.
