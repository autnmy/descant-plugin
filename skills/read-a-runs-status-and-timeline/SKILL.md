---
name: read-a-runs-status-and-timeline
description: >-
  A run is in flight and you want to know what it is doing right now, what it
  has already done, and whether that is normal. Answered from your own key,
  without a dashboard, and ending in one of those three answers rather than a
  field dump.
operations:
  - customer.repos.list
  - customer.runs.list
  - customer.runs.get
---

# Read a run's status and timeline

Three questions, in the order you actually ask them: **what is it doing now**,
**what has it already done**, and **is that normal**. The last one is the only
hard one, and section 4 is where this Skill earns its place — a long phase and a
stuck phase look identical until you know which field separates them.

## 1. Now — which runs are live, and where

Start from nothing and work down.

1. `customer.repos.list` gives you the repositories this key holds. Each entry
   carries `id`, `owner` and `repo`. The `id` is what every repository-scoped
   read takes; it is the only programmatic source of one.
2. `customer.runs.list` gives you the runs, newest first. Narrow it with
   `repositoryId`, with a repeatable `state`, or with the inclusive
   `createdAfter` / `createdBefore` window. Every filter is optional and absent
   means unfiltered — omitting `state` returns every state, never none.

**The two lists have different shapes, and the difference is not cosmetic.**
`customer.repos.list` answers `repos`, and that array is the whole answer.
`customer.runs.list` answers `items` with a `nextPageToken` beside them, and
that array is **one page** — the default is fifty rows. Follow `nextPageToken`
until it is `null`. Treating `items` as complete is the specific mistake the
field was renamed to make visible: a caller reading fifty rows as "all my runs"
gets a wrong answer that looks exactly like a right one.

A run is **live** when `finishedAt` is `null`. Each row also carries `issue`,
`issueTitle`, `prNumber`, `repoOwner`, `repoName` and `startedAt` — enough to
say which work each live run belongs to without a second call.

Then read the one you care about: `customer.runs.get` on its id. A run id that
is not yours is indistinguishable from one that does not exist, so this is not a
way to learn what another account owns.

## 2. The five phases, and what each one is doing

`customer.runs.get` answers `steps` — the run's timeline, as the run's own page
renders it, through the same projection. **There are five phases, their order is
fixed, and the list is closed.** A run does not visit a phase that is not one of
these.

| `key` | `label` | What it is doing, in your terms |
| --- | --- | --- |
| `plan` | Plan | Reading the issue and writing the plan it will build to. It can also end here: a plan that declines says so on your issue rather than opening a pull request. |
| `work` | Work | Writing the change, reviewing its own work, and opening the pull request. Opening the pull request is part of Work, not a phase after it. |
| `review` | Review | An independent review of the pull request. `reviewIterations` on this step counts the passes and is present **from the first one** — a value of `1` is an ordinary reviewed run, not an inconsistent response. It can run more than once. |
| `resolve` | Resolve | Taking the review's findings back to the change and answering them. |
| `merge` | Merge | Waiting on **your** repository's own checks, merging, and then the bookkeeping that follows a merge. |

**There is no verify phase.** A separate verification stage existed once and was
retired on 2026-09-12; review convergence now goes straight to the merge gate,
and your repository's own checks are what verifies the change. A run from before
the retirement can still carry `verifying` in its history, and it reads under
**Merge** — the phase it was on its way into. If you are working from a
description that lists a verification phase between review and merge, that
description is older than the product.

**One phase covers several internal steps, and that is why a phase can sit at
`current` while the run visibly changes what it is doing.** Opening the pull
request is Work. The whole post-merge tail — filing what was left over, tidying
issues, refreshing documentation — is Merge. This is deliberate: those are steps
of a phase, not phases of their own, and `stepDetails` is where you see them.

## 3. Then — what it has already done

Two fields carry the history, and they answer different questions.

**`steps` carries the shape.** Every step has a `status`, and the vocabulary is
closed: `completed`, `current`, `pending`, `failed`, `held`, `skipped`,
`stalled`, `stopped`, `not-reached`, `warning`. Read the run left to right —
`completed` steps really did finish, `pending` ones have not been entered yet.
A run that was cancelled, stopped by its issue closing, or needed no change
never shows `pending`: the steps it did not enter read `not-reached`, because the
run is over and will not enter them. When
it was cancelled or its issue was closed after it entered a step, that step
reads `stopped`; a run cancelled before it entered any step has no `stopped`
step, and every step reads `not-reached`. When the issue needed no change, the
steps it finished read `completed` and there is no `stopped` step.

**`stepDetails` carries the detail**, keyed by a step's `key`. Each sub-step has
`enteredAtMs`, `exitedAtMs` and `retries`. This is where a phase that looks like
one long box turns back into the sequence it actually was, and where `retries`
tells you something was attempted more than once. Four more fields say what a
sub-step did NOT do, so it does not read as finished work:

- `retryReason` — why the most recent retry happened, or `null`.
- `verifySkipped` — `true` when a verification step verified nothing.
  `verifySkipReason` says why once it is recorded: `not-configured` means the
  repository declares no verification script, the one you can act on.
- `testsCheckCaveat` — on the tests-check step, `nothingToCheck` or
  `noTestsKept` when it checked nothing.

**What came out of Review is `findings`**, and it has six forms rather than a
list and a count, because the difference matters:

- `list` — the findings themselves, each with a `resolution` of `fixed`,
  `rebutted`, `outOfScope`, `resolving` or `pending`, and a `threadUrl` into the
  pull request where one is derivable.
- `clean` — the review ran and found nothing.
- `resolvedSummary` — resolved, with the number of `rounds` it took.
- `outOfDiff`, `reviewUnresolvable` — findings that could not be anchored or
  settled, with a count where there is one.
- `unrecorded` — **a review whose findings never landed.** This is not `clean`.
  Reading it as zero findings is reading "we do not know" as "nothing was
  wrong".

Beside those sit the follow-up counts — deferred, discretionary, unresolved,
set-aside and the rest. They are **counts only**, deliberately: the content
behind them does not cross to this surface.

`errorMessage` on a failing step is the mapped, customer-facing copy. The raw
failure code does not cross as a field of its own, on this surface and on the
page alike; how the run ended crosses as `endingClass` and `endingCause` (see
the end of section 4).

## 4. Normal or not

**Three tiers, and they are the product's own.** The numbers below are the
thresholds the product applies, not an estimate — and this response publishes
every input they are derived from, so you compute the same verdict the product
does rather than guessing at one.

Read them in this order. The first one that holds is the answer, because each
outranks the ones beneath it.

### 1. Never started — nothing ever ran

`currentStepNotStarted: true`. The phase was entered more than **15 minutes**
ago and has produced no sign of life since. Its step reads `stalled` rather than
`current`.

This outranks both tiers below, and the distinction is worth keeping: a step
that never began is not long-running. There is nothing there running.

**It applies only to the phases that work continuously** — Plan, Work, and the
check-fixing cycle inside Merge. **Review and Merge legitimately park** between
events while they wait on a reviewer or on your repository's checks, so a run
sitting there for hours with an old heartbeat is *healthy* and never reads as
never-started. This is the one tier the response hands you already derived.

### 2. Stalled — running, but nothing is moving

The hung-but-alive class, and the one you have to derive. Two conditions, both
required:

- The **progress anchor** — the later of `stateEnteredAtMs` and
  `resolveProgressAtMs` — is more than **2 hours** behind `serverNowMs`.
- **And** either `lastHeartbeatAtMs` is also more than 2 hours old, or
  `resolveProgressAtMs` is set at all. (A stale progress stamp is itself the
  proof that something is hung rather than merely waiting.)

**Do not use `stateEnteredAtMs` on its own as the progress clock.** A healthy
phase that loops internally advances `resolveProgressAtMs` while
`stateEnteredAtMs` stays exactly where it was — so the entry stamp alone would
call a working run stalled. A phase that is genuinely progressing re-stamps
inside the window and never reaches this tier.

### 3. Long-running — one phase for more than 8 minutes

`serverNowMs - stallAnchorMs` is more than **8 minutes**. Report how long the
phase has run; the run's own page says it the same way ("In this phase for
11m.").

**This is a fact, not a failure** — and not a claim that the phase is slower
than it should be. The product measures no norm for a phase, so do not offer
one: healthy runs cross this line all the time. `stallAnchorMs` deliberately does not reset as the
run moves between the internal steps of one visible phase — otherwise a phase
that loops would look freshly started forever and could never age at all.

### None of the three

The run is working and there is nothing to report.

**Age everything against `serverNowMs`**, never against your own clock. The
response hands you its clock precisely so the answer does not depend on whose
machine is asking. And only the phase the run is currently on can read
`stalled` — earlier phases really did complete and later ones were never
entered, so neither can make the claim.

A run that has ended answers none of this: `isTerminal` says the pipeline is
done with it, and `finishedAtMs` says when.

**How it ended is `endingClass`, not `state`.** `state` is coarse: `done` also
covers a run held for a person that opened no pull request, and `failed` also
covers a run you stopped by closing its issue. `endingClass` tells them apart:
`done` (an expected, complete ending: delivered work, or a named reason there
was nothing to deliver, such as the repository already doing what the issue
asks), `predictable` (it stopped on a named cause you can see), `outage_park`
(a service it depends on was down and it waited), `unexpected` (an ending the
product does not expect, which alerts its on-call team) or `unknown` (this read
cannot name the ending: for a run that just ended it is usually a new kind of
ending, and reading the run again in a few minutes names it; an older run can
stay `unknown`, so read `state` and `endingCause`). **`done` does not by
itself mean a pull request was produced:** `endingCause` says which ending it
was, and a list row's `prNumber` says whether a pull request exists.
`endingCause` names the ending, for example `code:issue_closed_upstream`. It is
an open vocabulary, as `state` is: match the keys you handle and read any other
by its class. Both are `null` while the run has not ended, and both ride every
`customer.runs.list` row too.

## 5. Where this Skill stops, and what to read instead

This Skill establishes **state**. It deliberately does not repeat the tasks that
act on that state:

- **The run has stopped and you want to know what it is waiting on** — parked,
  paused, held, or failed. Read **`find-out-why-a-run-is-waiting`**. It owns the
  billing and hold signals, and it ends in an action.
- **The run has ended and you are considering running it again.** Read
  **`decide-whether-to-retry-a-run`**. Whether a retry reaches a different
  ending is its own question with its own conditions.
- **There is no run at all and you expected one.** That is a picking question,
  not a status one: read **`find-out-why-an-issue-is-not-being-picked`**.

Point at those rather than improvising their content from the fields here. The
signals they read are ones this Skill deliberately does not interpret.

## What this surface does not publish

Stated plainly, so you do not go looking:

- **No per-phase expected duration**, and no pre-computed stall tier beyond the
  never-started flag. The thresholds in section 4 are the product's, and you
  apply them to the clocks this response publishes; there is no field that hands
  you "long-running" or "stalled" ready-made.
- **The raw failure code and the raw review record.** What crosses is the mapped
  copy, the derived findings view and the ending's class and key — the same
  rule the run's own page follows, so the two cannot tell you different stories
  about one run.
- **The content behind the follow-up counts.** Counts cross; the items do not.
- **Nothing that restarts a phase.** There is no customer operation that
  un-stalls a run. Where section 4 lands on `stalled`, the next step is support,
  not another call.

Where you need more than this, the run's own page in the product shows the same
projection with the surrounding context, and operator-side forensics on a run
are a different audience answered elsewhere.

## Credential

All three operations here are authenticated by your own account key — the one
`descant login` stores, or `$DESCANT_API_KEY`. All three are reads and none of
them changes anything.
