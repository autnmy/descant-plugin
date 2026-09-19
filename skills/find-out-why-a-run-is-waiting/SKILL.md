---
name: find-out-why-a-run-is-waiting
description: >-
  A run started and then stopped moving. Work out whether it is waiting on YOU
  or on a limit clearing, and finish either by doing the one thing that releases
  it or by knowing that waiting is correct and roughly how long.
operations:
  - customer.repos.list
  - customer.runs.list
  - customer.runs.get
---

# Find out why a run is waiting

A run has stopped and nothing is happening. There are two completely different
reasons, they need opposite responses, and telling them apart is most of this
task:

- **It is waiting on a decision — a HELD run.** It is waiting for a person to
  do something. **It will not resume on its own, ever.** Waiting is the wrong
  answer here, however long you wait.
- **It is waiting on a limit clearing — a PARKED run.** Capacity, a spending
  cap, or a billing pause. It resumes by itself when the limit lifts, and doing
  nothing is the correct action.

Read both signals before you decide. They live in different fields and a run can
look identical from the outside.

## 1. Find the run, starting from nothing

You do not need a run id to begin.

`customer.repos.list` gives the repositories this key's account holds.
`customer.runs.list` gives that account's runs, **newest first**, in `items` —
each one carrying `id`, `issue`, `issueTitle`, `state`, `repoOwner`, `repoName`,
`startedAt` and `finishedAt`.

**`items` is one PAGE, not the whole history.** Fifty rows unless you ask for
another size, and `nextPageToken` is how you reach the fifty-first: it is `null`
on the last page rather than absent, so following it until it is `null` is one
check. Reading only the first page is how you conclude "nothing is waiting"
about a list you never finished — and the list is newest-first, so a run that
stopped moving a while ago is exactly the one that newer runs have pushed off
page one.

**There is no "only waiting runs" filter.** What narrows this read is
`repositoryId`, `state`, and the `createdAfter`/`createdBefore` window — all
optional, all absent meaning unfiltered. None of them keys on why a run rests,
so they shorten the walk but do not answer this question: you still decide per
row. A run still going has `finishedAt: null`; the ones worth looking at are
those with `finishedAt: null` that have not moved, plus any run whose
`dispatchWaiting` is set (step 2).

**A run with `finishedAt` set is not waiting — it ENDED.** That is a different
question, and step 5 says where to take it. Check this first: three of the four
branches below assume a run that is still in flight, and reading a finished run
through them invents a reason it does not have. Step 4 and step 6 are where a
run that stopped for good belongs.

Keep the run's `id`. Step 3 needs it.

## 2. Is it PARKED? Check that first, because it needs no action

Three fields answer this, two on the run and one on the response as a whole:

- `dispatchWaiting` — the reading to branch on. One of
  `waiting-for-slot`, `waiting-for-vendor-budget`, `paused-cap`,
  `paused-no-credits`, or `null` when the run is not waiting on any of these.
- `dispatchBlockedReason` — the raw signal beneath it, `slot` or
  `vendor_budget`, and `null` on a run that is not parked.
- `billing` on the list response — `pauseReason` (`balance`, `cap` or
  `operator`, or `null` when it is not established) and `outOfCredits`.

What each one means for you:

| `dispatchWaiting` | what is holding it | what you do |
|---|---|---|
| `waiting-for-slot` | the account is at its concurrency cap; other runs hold the slots | nothing — it starts when a slot frees. To go faster, finish or cancel another run |
| `waiting-for-vendor-budget` | a budget outside your account is exhausted | nothing you can do from here; it resumes when that budget refreshes |
| `paused-cap` | a spending cap was reached | raise the cap, or wait for the period to roll over |
| `paused-no-credits` | a balance pause — `outOfCredits` is `true` | add credit; nothing else releases it |
| `null` | nothing the dispatch reading covers — **read `billing` before concluding anything** | see below |

**A parked run is the system working.** It is not a fault and there is nothing
to fix. If `dispatchWaiting` is set, you are done: the answer is "it resumes
when the limit lifts", and only `paused-cap` and `paused-no-credits` have an
action you can take.

### `dispatchWaiting: null` does NOT mean "not paused"

This is the trap in this step, and it is silent. That reading covers a balance
pause and a cap pause — it has no value for an **operator** pause at all. A
tenant an operator paused therefore reads as `dispatchWaiting: null` while
`billing.pauseReason` is `"operator"`, and an operator pause does not clear on
its own the way a cap or a budget does.

So read `billing` on the list response in its own right, not as a footnote to
`dispatchWaiting`:

- `pauseReason: "operator"` — an operator paused this account. Nothing in the
  product releases it and no run will start until it is lifted. Contact support;
  do not wait, and do not go looking for a hold that is not there.
- `pauseReason: "balance"` or `"cap"` — the money story behind
  `paused-no-credits` / `paused-cap` above, and `outOfCredits` distinguishes
  them.
- `pauseReason: null` — **"not established", never "not paused"**: the value was
  absent or was one this deployment does not recognise.
- `billing: null` on the whole response — the billing read itself did not
  answer. That is **unknown**, not "unpaused". Treat it as a reason to re-read
  rather than as permission to move on to step 3.

## 3. Is it HELD? Read the run, and read the timeline

`customer.runs.get` on the run's `id` returns the run's own page as a snapshot:
`steps` (the canonical phases, each with a `status` and a mapped
`errorMessage`), plus the fields that say what it is waiting for.

A step's `status` is one of `completed`, `current`, `failed`, `held`, `pending`,
`skipped`, `stalled` or `warning`. **A step with `status: "held"` is the
signal** that something is waiting on a person.

Then read these three, in this order — they are not the same question:

- **`heldStepNote`** — the product's own sentence about the hold, and the thing
  to quote. It is non-null only when the timeline really carries a held step.
  Read it before deciding anything: it is the only place the specific reason is
  written in words.
- **`awaitingManualMerge`** — `true` when a finished run left a pull request
  open and a person has to deal with it. It tells you **a pull request needs
  human attention**. It does NOT tell you the merge will work: it stays `true`
  when the branch needs a rebase, and for every held Merge step regardless of
  what is blocking it. `heldStepNote` is what says which of those you are in.
- **`prNumber`** on the run summary — `null` when no pull request exists.

### Do not read "held" as "go and merge it"

A run can be held **before its pull request ever existed**, and it still carries
a held step. There is nothing for anyone to merge in that case, and treating the
held step alone as "merge it" sends you looking for a pull request that was
never opened.

So `awaitingManualMerge: true` means **there is a pull request and it is
waiting on a person** — read `heldStepNote` to learn what that person has to do.
Do not go straight to merging: the same `true` covers a branch that needs a
rebase first, a protection rule that has to be satisfied, and unresolved review
findings. Merging blind in those cases fails, or lands something that should not
have landed.

A held step with `awaitingManualMerge: false` — or with `prNumber: null` — means
something other than a pull request is waiting, and `heldStepNote` is again
where it is written.

### A held run never resumes by itself

This is the part worth being blunt about. A held run is finished as far as the
system is concerned; it admits no further events. No amount of waiting moves it.
Either a person does the thing `heldStepNote` describes — usually merging or
closing the pull request in your own repository — or the run stays exactly where
it is.

**Where you act is your own repository, not here.** Merging the pull request,
answering a review comment, closing the issue: those happen on the repository.
There is no customer operation that answers a hold, and this Skill does not
pretend otherwise. What it gives you is the run id, the pull request number and
the product's own sentence about what is being waited for.

## 4. Did it FAIL? Check that before calling it active

A run that failed is terminal, carries no held step, and has `dispatchWaiting:
null` — so it satisfies every condition for "still working" while being nothing
of the sort. Rule it out first.

- `state` on the run says `failed` when it did.
- The failing step in `steps` carries the mapped `errorMessage` — the
  customer-facing sentence, never a raw internal code.
- `finishedAt` on the summary is non-null for a failed run even though nothing
  "finished": the list read fills it in from when the run entered the failed
  state, precisely so a consumer cannot read a terminal run as still going.

If it failed, that is your answer: the `errorMessage` says why it stopped. What
to do about it — whether running it again is worth anything — is the retry
task's question, in step 5.

## 5. Is it none of those — just slow?

If it is not parked, not paused, not held and not failed, the run is still
working.
Measure how long the current phase has been going:

- `serverNowMs` — the server's clock when the response was built.
- `stallAnchorMs` — when the current customer-visible phase was entered.
- `currentStepNotStarted` — `true` when the current phase has produced no
  heartbeat yet, which is why it may render as stalled rather than current.

**Age the run as `serverNowMs - stallAnchorMs`, never against your own clock.**
The response hands you its clock precisely so the answer does not depend on
whose machine is asking.

A long `stallAnchorMs` gap with `currentStepNotStarted: true` is the shape worth
raising with support — it is not a hold and not a park, and this Skill has no
action for it. That is a real gap, not an omission: there is no customer
operation that restarts a stalled phase.

## 6. Or it already ended

`isTerminal` on the run says the pipeline is done with it, and `finishedAt` on
the summary carries when. A run that ended is not waiting on anything, so none
of the answers above applies to it.

Two of those endings still want something from you — a terminal run can carry a
held step, and `awaitingManualMerge` is only ever `true` on a run that has
finished. So read the hold signals in step 3 first, and treat "ended" as the
answer only when no step is `held` and `awaitingManualMerge` is `false`.

If it ended and you are here because it did not do what you wanted, the question
is whether to run it again — and that is its own task with its own conditions.
Read **`decide-whether-to-retry-a-run`**, which covers when a retry is worth
anything and when it repeats the same outcome. Do not improvise it from here.

## What NOT to do

- **Do not cancel a held run and start it again to "unstick" it.** A hold is
  waiting on a person, so a fresh run walks into the same hold. Cancelling and
  re-running is its own task with its own conditions —
  `decide-whether-to-retry-a-run` — rather than something to improvise here.
- **Do not act on a parked run.** Capacity and budget waits clear themselves;
  cancelling one just loses the work already done.
- **Do not age anything against your own clock.** Use `serverNowMs`.
- **Do not infer a reason the fields do not give you.** `pauseReason` is `null`
  for "not established", never for "not paused"; `billing: null` is "the read
  did not answer", not "nothing is wrong"; and `dispatchWaiting: null` does not
  cover an operator pause at all.
- **Do not merge just because `awaitingManualMerge` is `true`.** It means a
  pull request wants a person, not that merging will work. Read `heldStepNote`.

## Where this ends

One of six answers, and every one of them is a decision:

1. **Parked** — nothing to do, it resumes when the limit lifts; for `paused-cap`
   and `paused-no-credits` you can lift it yourself.
2. **Paused by an operator** (`dispatchWaiting: null` with
   `billing.pauseReason: "operator"`) — nothing releases it from inside the
   product. Contact support. Waiting is not an answer here.
3. **Held with a pull request waiting** (`awaitingManualMerge: true`) — do what
   `heldStepNote` says. That is often "merge it", and sometimes "rebase first"
   or "satisfy the protection rule" — the note decides, not the flag.
4. **Held with no pull request** — do what `heldStepNote` says, in your
   repository. It will not resume on its own.
5. **Failed** — `errorMessage` on the failing step says why it stopped. Whether
   to run it again is `decide-whether-to-retry-a-run`.
6. **Still running** — age it against `serverNowMs` and raise a long stall with
   support.

And if none of those fits because `billing` came back `null`, the honest answer
is **unknown** — re-read rather than guessing a verdict.

## Credential

Every operation here is authenticated by your own account key — the one
`descant login` stores, or `$DESCANT_API_KEY`. All three are reads and none of
them changes anything. A run id that is not yours is indistinguishable from one
that does not exist, so this is not a way to learn what another account owns.
