---
name: find-out-why-an-issue-is-not-being-picked
description: >-
  Work out why a repository is quiet — an issue you expected to be worked on has
  not been started — and either fix the cause or state which one it is.
operations:
  - customer.repos.list
  - customer.repos.poller-status
  - customer.repos.config
  - customer.repos.poller-diagnostics
  - customer.runs.list
  - customer.repos.update
---

# Find out why an issue is not being picked

You expected work to start on an issue and nothing happened. Several separate
causes look identical from the outside, and they are answered by different
reads. Work down the list; stop at the first one that answers.

Every step below is a read you can make with your own key. Read only what the
step's own result says — where a fact is not in the response, this Skill says
so rather than inviting you to infer one.

## 1. Find the repository, and check it is being polled

`customer.repos.list` gives every repository this key's account holds, with its
id. You need the id for every later step.

`customer.repos.poller-status` on that id answers "why is nothing happening".
Branch on `status.kind` — it is a union over sixteen causes, because "not
polling" has several different remedies and a single flag loses the difference.
`repo.paused` and `billing.paused` are the raw flags beneath that reading, and
`lastPoll` is the stored tick beneath both.

A repository the account paused is the most common answer and the cheapest to
fix: `customer.repos.update` un-pauses it.

A billing pause is NOT the same thing and `customer.repos.update` will not lift
it — read `billing.reason`, and note that `billing.degraded` means the billing
read itself was degraded, so `paused` and `reason` may be stale rather than
wrong.

**This response carries no polling interval.** Its fields are `status`,
`lastPoll`, `repo` and `billing`. After un-pausing, judge progress from
`lastPoll` moving, not from a wait you worked out here.

## 2. Check what the repository is configured to pick

A repository that is polling normally still picks only the issues its
configuration selects. `customer.repos.config` reports that selection: the
label or mode that makes an issue eligible, and the order the eligible ones are
taken in.

An issue missing the eligibility label is not a fault. It is the configuration
working, and the fix is on the issue, not here.

## 3. Ask what the picker would do right now

`customer.repos.poller-diagnostics` is a live read that answers what the picker
would do across the whole repository: how many issues carry the eligibility
label, how many would be picked, and which buckets hold the rest.

**It is a repository-wide read and takes no issue.** It cannot tell you where
one particular issue sits, or whether that issue is eligible. What it is good
for is the shape of the answer: a repository with eligible issues and none
picked is a different problem from one with nothing eligible at all.

Three things will mislead you if the response is read as a repository total:

- Branch on `configured` first, then on `mode`. Two of the three success shapes
  report **no counts at all** — `configured: false`, and `mode: "all-issues"` —
  because neither can be measured in label buckets.
- Check `truncated` before reading any count as a total. The counts are bounded
  by one page.
- When `customOrdering` is true, the buckets describe the default walk rather
  than what this repository would actually pick.

## 4. Check whether it already ran

`customer.runs.list` lists the account's runs, newest first. An issue that
looks untouched may have been picked and finished, or picked and stopped. A run
already exists for it in either case, and the question is then what that run
did rather than why nothing started.

This is also the only step here that can speak about ONE issue: match on the
run's issue rather than trying to make step 3 answer it.

## What this Skill will not tell you

Why a run that DID start behaved the way it did. That is a different task and a
different sequence of reads, and answering it by re-reading the poller is the
most common wrong turn here: the poller's job ended when the run began.

## Credential

Every operation above is authenticated by your own account key — the one
`descant login` stores on this machine, or `$DESCANT_API_KEY`. No other
credential is involved, and none of these reads changes anything except the
un-pause in step 1, which is a change you asked for.
