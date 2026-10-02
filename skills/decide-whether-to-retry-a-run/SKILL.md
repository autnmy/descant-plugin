---
name: decide-whether-to-retry-a-run
description: >-
  A run ended and you are considering trying again. Work out whether a retry
  reaches a different ending or reproduces the same one, what it costs, and —
  where it is the right move — start it and confirm it took.
operations:
  - customer.runs.list
  - customer.runs.get
  - customer.runs.cancel
  - customer.runs.restart
  - customer.repos.list
  - customer.repos.update
---

# Decide whether to retry a run

Trying again is cheap to type and not cheap to run. Some endings change on a
second attempt and some reproduce exactly, and the difference is readable before
you spend anything.

**So the first move is a decision, not a verb.** Work out which ending you have.
Only then does this Skill tell you how to act on it.

## 1. Read the ending

`customer.runs.list` gives the account's runs newest first in `items`, each with
its `id`, `state`, `finishedAt` and `prNumber`. Find the run you mean and take
its `id` — and note `items` is ONE PAGE, fifty rows unless you ask otherwise. A
run that ended a while ago sits behind `nextPageToken` rather than being gone,
so follow it before concluding the run you are looking for is not there.

`customer.runs.get` on that id is the run's own page as a read: the five phases
in `steps`, their `status` and `errorMessage`, and the flags that say what the
run is waiting on.

Three fields decide everything below:

- **`isTerminal`** — whether the run is over at all. A run still going is not a
  retry question yet; see step 2.
- **`awaitingManualMerge`** — a run that finished its work and is waiting for a
  person to merge the pull request it left open.
- **the failing step's `errorMessage`** — the mapped, customer-facing sentence
  for why a step ended the way it did.

**`errorMessage` is the whole reason available to you, and it is deliberately
prose rather than a code.** The operation states this: the mapped failure copy
crosses to this surface and the raw failure code does not. So read the sentence
and act on what it says. Do not look for a machine-readable failure field on
this response — there is not one, and a retry decision taken on a guess about
which code is underneath is exactly the mistake the next section is about.

## 2. Match the ending to an answer

The three endings below get three different answers. They are not
interchangeable, and the most expensive mistake here is giving the third one the
first one's advice.

### The run is waiting on a person

`awaitingManualMerge` is true, or `heldStepNote` is non-null.

**A retry does not fix this and will not make the wait shorter.** Nothing failed.
The run did its work and stopped at a point that wants a decision, and starting a
second run leaves the first one's pull request exactly where it was while paying
for the same work twice.

That is a different task with a different sequence — working out what a held run
is waiting for, and giving it that. Do that instead of retrying. Come back here
only if the answer turns out to be that the run genuinely ended badly.

### The run ended on a fault

The `errorMessage` describes something that happened *to* the run rather than
something about your change: a dependency that could not be reached, a step that
timed out, an upstream service that answered an error.

**This is the ending a retry exists for.** The condition may well be gone. Go to
step 3.

### The run ended because the change itself could not be got through

The `errorMessage` describes a limit the work itself ran into — the change was
too large to review in one piece, or the review could not conclude on it.

**A retry reproduces this ending, and this is the case where "just run it again"
is the wrong advice.** The same change presented the same way meets the same
limit, so a second run spends the same money to arrive at the same sentence. The
product's own position on this class is that the remedy is to make the change
smaller or to put a person on it — never to start the run again unchanged.

Split the work into smaller pieces and let those run.

**Whether there is anything to review yet depends on how far the run got, so
check before you send anyone to look.** `prNumber` on the run is nullable: a run
that ended while planning or implementing never opened a pull request, and
`null` there means there is no artifact to review and splitting the work is the
whole remedy. Where `prNumber` is set, asking a person to review what is already
there is the second option, and a retry would not have improved it either way.

## 3. What a retry costs, before you start one

A retry starts again at the first phase and works forward, so every phase the
first attempt completed is performed and billed again. There is no partial
re-drive that picks up where the last one stopped.

**The two ways to retry in step 4 are not the same run.** A restart in place
(`customer.runs.restart`, the path you can take yourself) re-runs the SAME run:
same id, and no second row appears in `customer.runs.list`. A fresh run, which
only a person can start, is a new run with its own id, and the original keeps
its row.

Two things worth doing first:

- `customer.runs.list` returns `billing` beside the runs. `outOfCredits` and
  `pauseReason` there tell you whether the account can currently start work at
  all — a retry into a paused account will not dispatch, and you will have spent
  the decision rather than the money.
- **Write down how the run ended before you restart it.** A restart in place
  reuses the run, so once it ends again its `state`, its steps' `errorMessage`
  and its `prNumber` describe the second attempt, and nothing on this surface
  keeps the first. Record the run id, the failing step's `key` and
  `errorMessage`, the `state` it ended in and its `prNumber` now. (A fresh run
  leaves the original's row as it was, so there the record stays readable.)

**What happens to the pull request the first attempt opened is not something
this surface states, so this Skill does not guess.** Note `prNumber` before the
retry, and read it again in step 5 — on the SAME run id after a restart in
place, or on the new run after a fresh one. If the number changed you have two
pull requests and the first is yours to close or keep; if it matches, the retry
continued on the same one. Either way you will know by reading rather than by
assuming, and an assumption here is how a customer ends up with an abandoned
branch they never looked for.

## 4. Start it

**Stop the old run first if it is still going.** Two runs working the same issue
at once is the one outcome worth ruling out, and it costs for both. If
`isTerminal` is false, call `customer.runs.cancel` on the run id and read the
`outcome` it answers:

- `cancelled` — stopped. Go on.
- `cancel_requested` — recorded, not yet stopped. **Re-read the run rather than
  calling again**; calling twice does not stop it sooner.
- `already_finished` — there was nothing to stop, and there never will be. Go on.
- `privately_dispatched` — the work left this estate and cannot be recalled from
  here. Do not start a second run against the same issue until you know what
  became of the first.

If the call answers `outcome_unknown`, the cancel **may** have landed. Re-read
the run before doing anything else; do not call again blind.

**Then start it, and know which of the two things you are doing.** The two paths
are not the same and they confirm differently:

- **Restart in place** with `customer.runs.restart` on the run id (the run's own
  page offers the same restart as a control). This re-enters the pipeline for
  the SAME run: no second run is created, and the run you already have changes
  state. It is accepted only for a run in `failed` state, and only for a key
  created by someone who is currently an owner of the organization. Read what
  it answers:
  - `accepted: true` with `awaitingSlot: false` — it started. Go to step 5.
  - `awaitingSlot: true` — recorded, waiting for a free run slot. It starts on
    its own when one frees; do not call again.
  - `conflict` — the run or its repository is not in a state that admits a
    restart right now: the run is not `failed`, its repository is paused,
    another run is already working the issue, a restart is already in
    progress, the platform has paused new work for a while, or the run's
    ending is one a restart cannot answer. It is not a promise that restarting
    can never work. Re-read the run with `customer.runs.get` and check the
    repository's `paused` flag on `customer.repos.list`. If the run is no
    longer `failed`, there is nothing to restart. If the repository is paused,
    it can be resumed with `customer.repos.update` on its `id` with the body
    `{ "paused": false }` — but that is a WRITE to the whole repository, not to
    this run: it starts work on every issue the repository picks, and someone
    paused it on purpose. Make it only when `paused` is true and resuming is
    yours to decide; it needs a key holding `repos:write`, and no owner role.
    Otherwise wait for the other run or the pause to clear. Then restart again —
    and if it is still refused after that, treat it as step 6 does.
  - `forbidden` — the key's creator is not an owner of the organization. An
    owner has to create the key, or do the restart; retrying with this key
    will never work.
  - `outcome_unknown` — the restart **may** have landed, and a second one is a
    second bill. Re-read the run (step 5) before doing anything else.
- **Start a fresh run** for the issue, from the new-run form. That is a
  person's step: there is no operation on this surface that creates a run, so
  for you the restart above is the path. A fresh run is a genuinely new run,
  and the original stays exactly where it is.

Prefer the in-place restart where it is accepted: it is the one you can take
yourself, and it keeps one run's history in one place.

## 5. Confirm it actually started

Do not treat having called or clicked as having started.

**Which check is the right one depends on which path you took in step 4**, and
running the wrong check is how a restart that worked gets reported as one that
did not.

**After an in-place restart**, there is no new run to look for. Re-read the SAME
run id with `customer.runs.get` and look for the run moving again:

- `isTerminal` is back to `false`,
- `state` has left the one it ended in,
- `stateEnteredAtMs` has advanced.

Do NOT go looking for a second row in `customer.runs.list` here, and do not read
`seededFromRunId` as a retry lineage — that field points at the source run whose
pull request a re-review run was seeded to review, and it is `null` on an
ordinary run. A restart in place sets neither.

**After starting a fresh run**, `customer.runs.list` is newest-first, so the new
run is at the top: match it on `issue` and check `startedAt` is after the
original's `finishedAt`. The original keeps its own row unchanged.

Either way, if nothing moved at all, check the `billing` block from step 3
before trying again — a paused account is the most common reason a start does
nothing.

## 6. Stop at two

**Two identical retries that end identically are evidence, not bad luck.** If a
second attempt reaches the same ending with the same `errorMessage`, the
condition is not transient and a third attempt buys nothing but the bill.

Stop there and escalate with the four things that make the report actionable:

- the run ids — after a restart in place that is ONE id, restarted, and you
  say so; after a fresh run, both ids,
- for each attempt, the failing step's `key` and its `errorMessage`, verbatim —
  after a restart in place, the first attempt's are the ones you wrote down in
  step 3, because the run now shows the second,
- the `state` each attempt ended in,
- the `prNumber` each attempt left, so whoever picks it up can see what was
  left behind.

A fault that reproduces exactly is a platform problem rather than a problem with
your change, and it is worth reporting as one.

## Credential

Every operation above is authenticated by your own account key — the one
`descant login` stores on this machine, or `$DESCANT_API_KEY`. No other
credential is involved. The three reads change nothing. `customer.runs.cancel` stops
a run. `customer.runs.restart` starts a failed one again and is billed like any
run, so it needs the key's creator to be an owner of the organization.
`customer.repos.update` changes a whole repository's settings and needs a key
holding `repos:write`.
