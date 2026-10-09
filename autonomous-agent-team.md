# The hard part isn't making agents act

*Shubh Bhagya — nine agent roles that research, build, verify and merge against a
live site, with nobody watching. Working notes, gathered while building it
2026-10-07 to 10-09. Not finished.*

> **Status: notes, not a case study.** Everything below is measured or quoted from
> a real run. The narrative shape comes later; right now the job is to not lose the
> evidence. Dates and run IDs are kept so every claim stays checkable.
>
> The shape did start to emerge on 10-09, and it is not the one the first draft
> assumed. The interesting material is not whether agents can do the work — that was
> settled on day one. It is the **eleven handoffs after the work is done**, nine of
> which were silently broken, and the two days that cost. See *The harness is the
> project* and the three set-pieces listed under *Pointers*.

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

Two more days of running it sharpened that into something more specific, and more
useful.

**The agents were never the hard part. The handoffs were.** Ranking, validating
against production, designing, building, QA'ing, labelling, gating, merging — all of
that worked, unattended, early. What did not work was everything *after* the merge:
verifying the deploy, proving which commit was live, closing the issue. The loop got
to its last mile and stopped there, nine times, for nine different reasons, and **not
one of them produced a red run anybody read.**

The number that makes the point: in the two days after the system started merging its
own work, **three commits touched the product and thirty-two touched the plumbing.**
Not because the backlog was empty — 54 product issues sat open — and not because the
agents were weak. Because a chain of eleven handoffs with a silent break in nine of
them takes two days to find, and you cannot find them by reading the code.

So if there is one sentence: **an autonomous system is not an agent, it is a chain of
handoffs, and every handoff is a place for a green tick to mean nothing.**

Everything else in these notes is downstream of that.

---

## The harness is the project — measured, 2026-10-09

Two days in, with the loop running unattended and merging its own work:

| | commits on `main`, 2026-10-07 to 10-09 |
|---|---|
| touched `src/` — the deployed application | **3** |
| plumbing, workflows, scripts, docs | **32** |

Of twenty merged pull requests, **three** changed the product. The rest were the
agents repairing the machinery that lets agents work.

That ratio is the finding, not an aside. The backlog was never the constraint — 54
product issues sat open against 12 infrastructure ones. Nor was it agent capability:
the cycles that *did* reach product work ranked, validated against production, built,
QA'd, labelled and merged without a human in the chain. **What consumed the two days
was that the loop could not finish.** It merged and then stopped, every time, for a
different reason each time.

So the honest shape of a project like this is not "build agents, then they work". It
is: build agents, discover the last mile does not exist, build the last mile, and
discover it has nine separate breaks in it.

Worth being precise about the counterfactual, because "we wasted two days on
plumbing" would be the wrong lesson. Two of those plumbing fixes were not optional:
one stopped CI from spending the product's entire daily LLM budget (below), and one
was the only reason any issue could ever close. The work was necessary. **What was
avoidable was not knowing it was necessary** — none of the nine defects announced
itself.

---

## Nine defects in the loop's own plumbing. Not one produced a red run anybody read.

Found in a single day, by tracing why one correct, QA-passed pull request would not
merge.

| what was broken | how long | what hid it |
|---|---|---|
| `agent-merge` read `steps.pr.outputs.sha` **inside the step whose id is `pr`** — always empty, `HTTP 422`, exit 1 | since written | the other trigger leg worked, so merges happened; this leg failed 100% of the time, invisibly |
| `close-on-evidence` lacked `pull-requests: read`, so its close step got `HTTP 403` | since written | the job's `if` had never once passed, so the step had never run |
| nothing ever ran `gh pr ready`, so agent PRs stayed drafts | since the convention began | a draft cannot merge, and GitHub reports nothing about why |
| `pr-gates` omitted `ready_for_review` from its trigger `types` | since written | so a readied draft had **no checks at all** — and the merge gate refuses on "no checks reported", a correct rule applied to a PR that could never satisfy it |
| `agent-merge` fetched `isDraft` into a file and never read it | since written | masked by the four above |
| the closing keyword was prose nobody was required to write | since written | the first successful `close-on-evidence` run in the repo's history closed **nothing**: `PR #312 claims no issue` |
| a `GITHUB_TOKEN`-created push triggers no workflow | platform behaviour | agent-merged commits were never verified against production at all |
| a `GITHUB_TOKEN`-*dispatched* run emits no completion event either | platform behaviour | the verification then ran, went green, and nothing consumed it |
| `agent-work` lacked `actions: read`, so its measurement fetch got `HTTP 403` | since the loop was wired | the step is `continue-on-error`, so it printed the 403 and the run went green |

The last one is the worst, and not because of the 403. **The delivery cycle had never
once received the analyst's measurement.** The prompt said "read the report, if
present"; it was never present. That is the arm the architecture calls load-bearing —
*measurement decides discovery, discovery feeds delivery, delivery produces evidence,
measurement scores it* — and it fed nothing into delivery for its entire life while
every run reported success.

Four of these are invisible in the YAML and only visible in whether the thing ever had
an **effect**. A static gate cannot see them. That is the argument for a report that
asks "did this leg ever succeed", "did this workflow ever close anything", "did the
cycle receive a measurement" — and that distinguishes *produced nothing because there
was nothing to do* from *produced nothing because it is broken*. Conflating those two
is precisely what hid the 403 for its whole existence.

---

## The platform fact that dominated everything: GITHUB_TOKEN's recursion guard

One rule, bitten three times at increasing depth, each time costing a day's confusion.

GitHub does not start a workflow run for an event created with the default
`GITHUB_TOKEN`. The documented exceptions are `workflow_dispatch` and
`repository_dispatch`.

**Depth 1.** The merge workflow merges with `GITHUB_TOKEN`, so the push it creates
triggers nothing. Measured by comparing merges:

```
PR #299  merged by app/github-actions  ->  0 push-triggered runs
PR #292  merged by a human             ->  2
PR #296  merged by a human             ->  2
```

Consequence: no autonomously-merged commit was verified against production. On a site
with no staging, the gate the whole loop was built to end with simply was not running.

**Depth 2.** Fixed by having the merge workflow *dispatch* the verification — exempt,
so it starts. But the run it starts is still marked `GITHUB_TOKEN`-originated and its
**completion** reaches no listener. Same commit, two routes:

```
dispatched by the workflow (GITHUB_TOKEN)  run 37887912818  success  ->  nothing fired
dispatched by a user token, same commit    run 37890955328  success  ->  fired immediately
```

So the chain reached *merge → deploy → verified against production* and stopped there.
The verification happened and nothing consumed it.

**Depth 3.** A separate instance of the same family: "Allow GitHub Actions to create
and approve pull requests" was off, so the fallback that exists to rescue a cycle which
pushed a branch but died before opening its PR could not open one. It pushed the
branch, failed, and the work sat until a human opened the PR.

The general lesson is not about GitHub. It is that **an autonomous loop is a chain of
platform affordances, and the platform's anti-recursion defaults are aimed exactly at
what you are trying to build.** Every hop that a human would make by clicking is a hop
the platform may refuse to a token. The fix that worked was to **dispatch forward**
rather than react backward — a dispatch is exempt, so calling the next workflow works
where listening for the previous one does not.

---

## Verification that consumed the thing it was verifying

The production smoke sweep probed every URL in the sitemap. Twenty-four of those URLs
**generate** an LLM reading on a cache miss, at ~3,330 input tokens each.

The sweep ran on every push to `main`. On 2026-10-08 that was twelve times.

```
requests to /rashifal/ that day, by caller
  curl (CI)   980      57 of them succeeded
  browser       2       0
  bot          18      18

57 successful CI generations x ~3,330 tokens  =  ~190,000
Groq free tier, per model, per day            =   200,000
```

Then, from the production logs:

```
tokens per day (TPD): Limit 200000, Used 198298, Requested 3779
AllProvidersExhausted: All 2 provider(s) in chain failed
```

**21 of 24 rashifal pages were serving 503 to real users**, because a health check had
spent the product's entire daily budget proving that pages render. Two browser requests
arrived during the window; both got the error page.

Three things compounded, and each is worth keeping:

- The pacing fix made it worse. Throttling the sweep to stay under the *per-minute*
  ceiling capped the token **rate** and not the **spend**, and made the sweep so slow it
  began timing out — thirteen consecutive runs died at exactly 600s.
- A timing-out sweep **masked the checks behind it.** The gate exits non-zero, so every
  later assertion was skipped, including the only one that reaches the hardest surface
  on the site. The expensive step was hiding the valuable ones.
- The retry ladder multiplied the cost of a *failing* page. In-probe retry, serial
  confirmation pass, settle, final pass — a broken generating route is probed four
  times, and every probe is another generation attempt. 980 requests against a model
  predicting 288.

And the architecture document had predicted the wrong failure mode: it said the symptom
would be *a bill*, because the provider chain falls through to a paid leg. The chain in
production had two providers and no paid leg, so the symptom was **an outage**. A
documented risk, correctly identified, with the consequence guessed wrong.

---

## The CI minute allowance is the real scheduler — and it runs out silently

The repository is **private on the free tier: 2,000 Actions minutes a month.** That
number, not the model quota, turned out to govern how often an autonomous system can
run at all.

Month-to-date on 2026-10-09, nine days in:

| workflow | minutes | runs | per run |
|---|---|---|---|
| PR gates | 460 | 56 | ~8 |
| production verification | 404 | 34 | ~12 |
| **the agent delivery cycle** | **248** | 13 | ~19 |
| committed-CSS build check | 113 | 74 | ~1.5 |
| the six other agent loops | 87 | 83 | ~1 |
| **total** | **1,312 of 2,000** | | |

**The agents are the cheap part.** Verification costs 3.5x what the work costs: gates
plus production checks are 864 minutes — 66% of everything — against 248 for the
cycles that actually rank, design, build and QA. That is worth saying plainly, because
the intuition runs the other way. A reasoning agent writing code for twenty minutes
feels expensive; a test suite you run on every push, four times a day, with a mutation
harness inside it, is where the budget actually goes.

### The arithmetic that matters, and why it bites

688 minutes left with 22 days to go is **31 minutes a day**. The observed rate after a
day of optimisation was ~162. The worst single day was **552** — more than a quarter
of a month's allowance in 24 hours.

So the loop was running at roughly five times its sustainable rate, which means the
allowance is not a background constraint. It is the binding one, and it has a hard
edge:

**When the minutes run out, workflows do not fail. They do not start.** No run, no
check run, no annotation, no email. A pull request simply sits with nothing attached
to it, and a merge gate that requires "all checks green" sees zero checks and refuses
— correctly, and for a reason nothing in the repository explains.

That is the thesis again, arriving from a completely different direction: **the
platform's own exhaustion mode is silence that reads as nothing happening.** Every
other silent failure in these notes was something we built. This one ships with the
product.

### What actually moved the number

Two changes took the production check from ~19 minutes a run to ~3, and they generalise:

- **Do not verify what cannot have changed.** The check ran on every push to `main`. Of
  25 consecutive commits, **none** touched a path that ships — all were workflows,
  agent definitions, scripts and reports. The fix is a path filter derived from the
  Dockerfile's `COPY` lines rather than guessed, written as an ignore-list so a new
  top-level directory defaults to *being verified*. Fail-safe direction matters more
  than brevity here.
- **Do not let one expensive step hide the cheap ones.** The smoke sweep was 10 of the
  19 minutes and had started timing out; everything behind it was skipped. Removing
  the 24 generating URLs — the same fix as the outage story — took the sweep to 1.6
  minutes and let the most valuable assertion in the file run for the first time in
  thirteen runs.

Earlier, parallelising the mutation harness took it from 281s to 86s with the same
50/50 mutations caught. It was 61% of every gate run. **Slow because thorough, not
because wasteful** — so the answer was concurrency, not trimming coverage. Worth
keeping as the counterexample to "cut the slow test".

### The lever not pulled, and why it is interesting

**A public repository gets unlimited Actions minutes.** The entire budget problem is a
consequence of the repo being private, and flipping it would also unlock branch
protection — which this design wanted for the merge gate and could not have on a
private free repo.

It was not flipped, and the reason is a good illustration of how these decisions
actually go — including how easy it is to overstate one.

Passive discovery of an unpromoted repository is near zero, so the exposure was never
attention. A secrets scan over the tree and the last 400 commits found nothing: no
keys, no tokens, no `.env`, only `.env.example`. What it did find was one book
committed into `data/` despite `data/` being gitignored — `.gitignore` does not
retroactively untrack, so a file added before the rule stays tracked forever.

**My first write-up of this called it "a copyrighted textbook in git history" and
treated it as a serious exposure. That was wrong, and the repo's own registry said so
forty lines away:**

```python
Book("vedic_astro_textbook.pdf", "P.V.R. Narasimha Rao", ...,
     "https://www.vedicastrologer.org/articles/vedic_astro_textbook.pdf",
     "Author-distributed on his own site."),
```

The file is distributed free by its own author and fetchable by anyone. The entry
immediately above it reads *"In print. Commercial title — supply your own copy"* — the
one that genuinely cannot be redistributed, and the one that was never tracked. The
registry had the distinction right the whole time; I read "PDF in git" and supplied the
alarming interpretation myself.

So the honest version: redistributing an author-distributed PDF inside a repository is
not clearly licensed, removing it is correct, and it is **licence hygiene rather than a
leak**. It is still a precondition for going public, because the file stays retrievable
from history until that is rewritten — and the rewrite is not small: the commit is 43
in, so 686 of 729 commits get new SHAs, with every open PR and live agent branch
needing a rebase onto them.

Two general points, and the second is the one I would actually keep:

**A cost ceiling and a visibility decision turned out to be the same decision.** Public
means unlimited minutes *and* the branch protection this design wanted and could not
have. The obstacle was neither cost nor visibility but a file nobody remembered
committing.

**And writing up a risk is where you discover you exaggerated it.** The overstatement
survived being said out loud, written into notes, and pushed to a public repository. It
did not survive going back to check what the file actually was. A piece about silent
failure should include the author's own unexamined claim sitting in it for a day.

---

## A test harness can lock a bug in place

The static gate over the workflow files shipped with a mutation harness — it
re-introduces each defect in a temp copy and requires the gate to catch it. Good
practice, and this repo's standing rule: *prove a gate can fail before trusting that it
passes.*

Independent QA found the harness **requiring a false positive**.

The self-reference check matched `steps.<id>.outputs` anywhere in a step. But GitHub
interpolates only inside `${{ }}`, so a bare mention in a shell string is inert text —
and the gate failed on prose of exactly the kind these files are full of:

```
echo "see steps.live.outputs.ok in the next step"      ->  FAIL
```

One of the harness's own mutations planted that uninterpolated form and **required exit
1**. So the harness would have passed forever while holding the wrong answer, and fixing
the gate would have surfaced as a harness regression. That is the exact inverse of what
the harness exists to do.

It got sharper. Narrowing the check to require `${{ }}` was written as a character class
— `\$\{\{[^}]*steps\.<id>\.outputs` — which cannot cross a `}`. Neither a later `}`,
which was the intent, **nor one inside the same expression**, which was not:

```
${{ format('v{0}-{1}', github.run_number, steps.tag.outputs.sha) }}

  the version before the fix   exit 1   caught it
  the version that shipped     exit 0   missed it
```

A real instance of the original defect, walked straight through. No mutation covered
that shape, so the harness reported a clean 18/18 while the check had been weakened.
Found by QA diffing the two gate versions against one tree — not by the harness, and not
by the author.

**So: a mutation suite proves a gate can fail. It does not prove the gate is asking the
right question, and it will happily pin the wrong one.** Every mutation needs a control
on the other side of the boundary, and when a check is narrowed, the narrowing needs its
own mutation or it is unmeasured.

---

## The irony cluster, which is the chapter heading for all of it

Three in two days, and they are not jokes — each is the same structural mistake:

**An issue was closed by the commit that fixed issue-closing.** The commit titled
*"`Refs #N` must suppress the branch-name fallback, or it closes the issue anyway"*
contained the explanatory line *"would have closed #313 on merge"*. GitHub's
commit-message parser read `closed #313` as a claim and shut the issue. Nobody noticed
for ten hours — including a comment written on that issue saying "this stays open". The
fix was correct. **It hardened PR bodies, and GitHub parses commit messages with a
separate parser that no workflow reads.** Two things can close an issue; one got
hardened.

**The sweep that checked production took production down.** Above.

**The step that fetched the measurement reported success for its entire life, having
never fetched one.** Above.

The common shape: *the mechanism was right and its scope was wrong, and nothing
measured the scope.* A rule that covers one of two paths reads exactly like a rule that
covers both.

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

- **Open on the ratio: 3 product commits against 32 plumbing commits.** It is
  concrete, it is surprising, and it earns the rest of the piece in one line. Then
  explain *why*, which is the silent-success argument. Opening on the abstraction and
  arriving at the number is the weaker order.
- **The spine is the last mile, not the agent.** Readers expect "can an agent write
  code" and that question was settled on day one. The piece is about what happens
  between a merge and a closed issue, and that nobody writes about it because nobody
  gets that far.
- The honest arc is *every fix found by running it*. None would have been caught by
  reading the YAML — which is also why the loop validates against production instead
  of trusting an issue description.
- **Three set-pieces carry it, in this order.** Each is self-contained, each has a
  measurement, and each lands a different blade:
  1. *The health check that took the site down.* 980 CI requests, 190k of a 200k
     daily token budget, 21 of 24 pages serving errors to real users. Verification
     eating the thing it verifies. The most vivid, so it goes first.
  2. *The recursion guard, three levels deep.* The platform's anti-loop defaults are
     aimed exactly at what you are building, and the fix is counterintuitive —
     dispatch forward, never listen backward.
  3. *The harness that required a bug.* A mutation suite pinning a false positive as
     correct, so fixing the gate reads as a regression. This is the one a technical
     reader will not have seen before, so it goes last and lingers.
- **The budget section is this study's tie into the collection's through-line** — every
  system here runs against a ceiling that cannot be raised by spending. Lead with the
  counterintuitive split: verification costs 3.5x the work. Then the hard edge, which
  is the thesis arriving from outside: **when the minutes run out, workflows do not
  fail, they do not start** — no run, no check, no error, and a merge gate that
  correctly refuses a PR with zero checks for a reason nothing in the repo explains.
  Every other silent failure in the piece is one we built; that one ships with the
  platform.
- **The public-repo lever is the right place to end the budget thread**, because the
  answer is not a cost argument. Public means unlimited minutes *and* the branch
  protection this design wanted and could not have. The blocker turned out to be a
  copyrighted textbook sitting in git history, committed before `data/` was ignored —
  `.gitignore` does not retroactively untrack. A cost ceiling and a visibility
  decision were the same decision, and what stood in the way was neither.
- **Let the ironies do the closing argument, not a conclusion.** An issue closed by
  the commit that fixed issue-closing; a sweep that broke production while checking
  it; a step that reported success for its whole life having never once run. They are
  the same structural mistake three times — *the mechanism was right, its scope was
  wrong, and nothing measured the scope* — and stating that once, after the three
  stories, is stronger than a summary.
- **Keep the author's own errors in.** A force-push that destroyed work and closed a
  PR; a "20/20 gates pass" that was really 20 pass 1 fail; telling the user their
  daily quota was gone until tomorrow when it was a rolling window that refilled in
  ten minutes. A piece about silent failure that hides the author's own is making the
  mistake it is describing. They are also the most relatable paragraphs in it.
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
- **The eleven-handoff chain, with the nine breaks marked** — rank, validate, design,
  build, QA, label, gate, merge, deploy, verify, close. Green up to `merge`, and nine
  silent breaks after it. This is the piece's central image and probably its opener:
  it shows at a glance that the work everyone writes about is the left half, and all
  the difficulty is in the right.
- **The recursion guard at three depths** — push blocked, dispatched-run completion
  blocked, PR creation blocked. Three arrows stopping at the same wall, with the
  working path (dispatch forward) drawn around it.

### Things to check before publishing

Answered since the first draft:

- **Actions-minutes burn: measured properly, and it is a live constraint rather than
  a footnote.** 1,312 of 2,000 consumed in nine days, leaving 31 min/day sustainable
  against an observed ~162 and a worst day of 552. Verification is 66% of it and the
  agent cycles 19%. The two fixes that moved it are in the budget section; the open
  question is whether a loop this expensive is viable on a private free repo at all,
  or whether going public is a precondition rather than an option.
- **Whether a full unattended run closes its own issue: yes, once.** `close-on-evidence`
  closed an issue on production evidence for the first time in the repo's history —
  and only after the nine breaks above were fixed. Do not overclaim this: the trigger
  still came from a dispatch, which is the one link left.

Still open, and the piece should not pretend otherwise:

- **Whether a night runs unattended end to end, including the close.** Every
  successful close so far has had a human dispatch something. The fix for the last
  link is written and has never fired on its own.
- **Whether the measurement arm now actually feeds delivery.** The permission is
  granted and the 403 is understood, but no scheduled run has yet successfully
  fetched a measurement. Until one does, "the circuit is closed" is a claim about the
  code, not the system.
- **Whether the weekly ranking is consumed by the nightly cycle, or merely produced.**
  Carried over, still unanswered — and sharpened by a related finding: for two days
  every cycle was hand-dispatched with an explicit issue number, so the ranking was
  bypassed by the operator rather than ignored by the loop. That is its own small
  lesson about autonomy: **the human who keeps choosing the work is a link in the
  chain too**, and the easiest one to miss.
- Whether a report that asserts on *effect* rather than status actually catches the
  four defects a static gate cannot see. It is specified and being built; the
  validation is to run it against this two-day window and see.
