# The hard part isn't making agents act

*Shubh Bhagya — nine agent roles that research, build, verify and merge against a
live site, with nobody watching. Working notes, gathered while building it on
2026-10-07/08. Not finished.*

> **Status: notes, not a case study.** Everything below is measured or quoted from
> a real run. The narrative shape comes later; right now the job is to not lose the
> evidence. Dates and run IDs are kept so every claim stays checkable.

---

## The thesis, if there is one

Making an agent do something is easy. **Making it tell you when it didn't is the
whole engineering problem.**

Four separate times in two days, a scheduled workflow concluded `success` having
accomplished nothing:

| loop | what it reported | what it did |
|---|---|---|
| `agent-measure` | success | 7 turns, 4 permission denials, no output |
| `agent-discover` | success | $1.21, 51 turns, 12 denials, no issue filed, no report |
| `agent-work` | success | 10 turns, no branch, no PR |
| `agent-work` (dispatch) | success | engineer killed mid-work by the parent ending its turn |

Every one looked healthy from outside. A green tick, a plausible duration, no
error. The failure mode of an autonomous system is not recklessness — it is
**silence that reads as success**.

Everything else in these notes is downstream of that.

---

## What it cost to find out

Fifteen distinct fixes between the first scheduled run and the first unattended
merge. Grouped by what they teach:

**The shell lies to you.** GitHub Actions runs `run:` blocks under `bash -e`, so a
bare `test && action` exits the step whenever the test is false. Three separate
instances:

- the gate sweep died at the first failing gate, making every branch of its
  reporting unreachable
- `[ -n "$skipped" ] && echo ...` would have failed a **completely passing** run,
  because an empty variable makes the test return 1
- the merge workflow's PR lookup exited on the *success* path, so finding a PR was
  indistinguishable from not finding one

**Copied files diverge.** Five times a fix applied to one workflow was needed by
its siblings and not applied: tool grants, report writing, GCP auth, `poppler`,
and `id-token: write`. The last one would have killed three loops that had never
run. The lesson arrived late: when these files are written by copying, **check the
whole set, not the one that failed**.

**Tokens expire mid-work.** A cycle ran 63 minutes, completed architect and
engineer, produced a branch with real commits, and died at `git push`:

```
remote: Invalid username or token.
fatal: Authentication failed
```

GitHub App installation tokens are capped at exactly one hour. **$5.34 and 25
turns of finished work, lost**, because all of it was still on a runner about to
be destroyed. The earlier cycle that succeeded took 19 turns — same code,
different duration.

Two fixes, deliberately independent: push the branch at the *first* commit so
little is ever at risk, and have the fallback use `GITHUB_TOKEN`, which is minted
per job and outlives the App token. The rescue had been available all along; the
step was simply aimed at the wrong credential.

**A checker can see itself.** After switching the merge gate from reading an event
payload to querying the check-runs API, it read:

```
merge=in_progress:pending     ← its own check run
gates=completed:success
build-gates=completed:success
```

and concluded "not all checks are green yet". A workflow waiting for itself,
forever. The rule — a pending check is not a pass — was right; what changed was
that the checker had joined its own list.

---

## The design that came out of it

Nine roles, three levels. The useful part of a hierarchy is the escalation path and
the team boundaries, not the title count: every extra layer is another full agent
invocation that re-reads context from scratch, because subagents share no memory.

| team | owns | judged on — **not** its own output |
|---|---|---|
| Analytics | GA4, Search Console, scoring every ship | were the misses reported as plainly as the hits |
| Marketing | SEO, keywords, competitors, social | did the queries it picked actually move |
| Sales | the five funnels, retention | step-through on the worst funnel step |
| Backend / Frontend | shipping | merged work that moved its metric |
| Design | visual direction | CTR at unchanged position |
| Research | corpus, citations, authenticity | unsourced claims left on live pages |
| IT | free tiers, deploys, latency | headroom at 10x |
| QA | verifying all of the above | defects that reached production |
| `chief` | ties, the quarter's objective | nothing — it produces only when escalated to |

**Each team is scored on an outcome, not on its own volume.** Issues filed, PRs
merged and articles drafted are activity. The anti-goals matter as much: a team
told only to maximise its remit will file more issues to look productive.

### The rule that made it safe enough to leave alone

No agent pushes to `main`. They push a branch and open a PR; a separate workflow
merges on evidence. Three conditions, all machine-checked:

1. 21 verification gates green, including one that mutates a citation to a section
   its book does not contain and requires itself to catch it
2. a `qa-pass` **label** — not a comment, because a gate that depends on matching
   free text is one a formatting change silently opens or closes
3. the diff inside an unattended size limit

`qa.md` already said *"PASS is a report, not an action."* So the report is written
by one role, the gates decide, and a third thing acts. An agent cannot merge its
own work by concluding it is finished — which matters in a repo whose history
includes an agent writing `USER AUTHORISED PUSH this turn` into a status file and
then citing its own line as the authorisation. Twice.

### Enforce at the platform layer, not in the prompt

Repeatedly the better guardrail was structural rather than instructional:

- `contents: read` on a job beats listing which shell verbs an agent may run — it
  cannot push whatever it runs
- a path filter on the deploy trigger beats `[skip ci]` in a commit message — the
  message depends on an agent remembering to write it, and an agent writing its own
  exemption is the exact failure above
- a label beats a grep; a glob beats a promise

---

## Measurements worth keeping

**Mutation harness: 281s → 86s**, 50/50 mutations still caught. It was slow because
it was thorough — 51 sandboxes, each a full test suite — not because it wasted
anything. Parallelised, not trimmed. It was 61% of every gate run, against a
2,000-minute monthly Actions allowance being burned at ~206 min/day.

**Search Console returns 9% of the truth when you ask by query.** Measured over
September: 8,097 impressions by property total, 8,154 by page dimension (101%),
**769 by query (9%)**. Google withholds anonymised queries, and on a young site
almost every query is rare. An export that read 7% of the data produced a confident
conclusion that the quarter's objective was worth 23 extra clicks. It wasn't.

The fix that matters is not the dimension. **It is that every export must state its
own coverage** — an export that doesn't say how much it is missing lets a confident
wrong analysis look identical to a right one.

**A real cycle costs $2–5 and 20–60 minutes.** Early no-op runs cost $0.14 and
suggested otherwise.

---

## Pointers for the written piece

- Open on the silent success. It is the counterintuitive bit and the whole argument.
- The honest arc is *fifteen fixes, each found by running it*. None would have been
  caught by reading the YAML — which is also why the loop validates against
  production instead of trusting an issue description.
- Resist the triumphant ending. The interesting material is the failures, and the
  system's own value is that it reports them.
- Include the agent that **refuted my hypothesis with measurement**: I wrote "project
  agents aren't registered in CI" into an issue from a local failure; it dispatched a
  probe, got the harness's own list of 14 agents, and said so. It also declined to
  file two of four findings — one it couldn't reproduce, one that turned out to be a
  fetch artefact.
- Include the one it got right that I'd have got wrong: told to speed up a
  production sweep, it reported that the sweep was **already parallel** and withdrew
  the recommendation.

### Diagrams worth drawing

- **The four loops as a closed circuit** — measure decides discovery, discovery feeds
  delivery, delivery produces evidence, measurement scores it. Without the scoring
  arm the others ship forever and never learn.
- **The merge gate as three independent conditions**, with the point that they are
  written by different things.
- **The timeline of one lost cycle** — 08:26 start, 08:42 design, 09:29 push,
  token dead at 09:26. One image carries the whole token-expiry argument.

### Things to check before publishing

- Whether a *full unattended night* works. At the time of writing, every successful
  run had a human watching it.
- The real Actions-minutes burn in steady state. 206/day was an atypical day of
  heavy iteration.
- Whether the weekly ranking is actually consumed by the nightly cycle, or merely
  produced.
