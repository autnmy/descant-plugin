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

A retry is a **new run**, not a resumption. It starts at the first phase and
works forward, so every phase the original run completed is performed and billed
again. There is no partial re-drive that picks up where the last one stopped.

Two things worth reading first:

- `customer.runs.list` returns `billing` beside the runs. `outOfCredits` and
  `pauseReason` there tell you whether the account can currently start work at
  all — a retry into a paused account will not dispatch, and you will have spent
  the decision rather than the money.
- The original run's own record does not go away. Its row stays in
  `customer.runs.list` with its `state`, its `finishedAt` and its `prNumber`, so
  a retry never costs you the evidence of what happened the first time.

**What happens to the pull request the first run opened is not something this
surface states, so this Skill does not guess.** Read `prNumber` on the original
run, and read it again on the new one in step 5. If the numbers differ you have
two pull requests and the first is yours to close or keep; if they match, the new
run continued on the same one. Either way you will know by reading rather than by
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

**Then start it from the dashboard, and know which of the two things you are
doing.** There is no operation on this surface that restarts or re-drives a run
— the key-authed verbs for a run are list, read and cancel — so this step is a
person's. The two paths are not the same and they confirm differently:

- **Restart in place**, from the run's own page. This re-enters the pipeline for
  the SAME run: no second run is created, and the run you already have changes
  state. The control is offered only for a run in `failed` state, only to an
  owner of the account, and only where the platform is configured for it — so if
  you are looking at a failed run and see no restart control, one of those three
  is the reason, and it is not something a retry decision can route around.
- **Start a fresh run** for the issue, from the new-run form. This creates a
  genuinely new run, and the original stays exactly where it is.

Prefer the in-place restart where it is offered: it is the cheaper of the two,
and it keeps one run's history in one place.

## 5. Confirm it actually started

Do not treat having clicked as having started.

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

- both run ids,
- the failing step's `key` and its `errorMessage`, verbatim,
- the `state` each run ended in,
- the `prNumber` of each, so whoever picks it up can see what was left behind.

A fault that reproduces exactly is a platform problem rather than a problem with
your change, and it is worth reporting as one.

## Credential

Every operation above is authenticated by your own account key — the one
`descant login` stores on this machine, or `$DESCANT_API_KEY`. No other
credential is involved. Two of the three reads change nothing; `customer.runs.cancel`
is the only call here that does, and it stops a run rather than starting one.
