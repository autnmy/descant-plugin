---
name: onboard-a-repository-and-confirm-it-is-working
description: >-
  Take a repository from "connected" to verifiably picking up work — check what
  is already attached, read the configuration back in plain terms, change what
  needs changing, and confirm the poller is actually running before you call it
  done.
operations:
  - customer.repos.list
  - customer.repos.labels.list
  - customer.repos.config
  - customer.repos.update
  - customer.repos.poller-status
  - customer.repos.poller-diagnostics
  - customer.repos.settings-audit
  - customer.audit.list
---

# Onboard a repository, and confirm it is working

Knowing a repository is picking up work is the task. It is the half that goes
wrong quietly: a repository can be connected, configured and completely idle, and
nothing about connecting it tells you so.

**Scope, stated first because it decides whether this Skill applies to you.**
Connecting a repository is NOT on this API — there is no customer operation that
attaches one, and every read below needs a repository id that exists only once it
is connected. Connection happens by installing access to the repository. This
Skill starts the moment that is done; everything from there — configure, verify,
prove it is live — is here.

Finish on one of two answers — **a green light**, meaning the poller ran and you
know what it saw, or **a named blocker**. Never on "the call succeeded".

## 1. Find the repository, and get its id

`customer.repos.list` gives every repository this account holds, each with the
`id` every later step needs. Match on `owner` and `repo`.

**If it is not in the list, stop — that is your blocker, and this API cannot
clear it.** The repository is not connected to this account. Connect it first;
nothing below works without an id, and no operation here can create one.

**You will not find a duplicate.** A repository is unique across the system by
provider, owner and name, so connecting one that is already connected updates the
record you have rather than adding a second, and binding a repository another
account holds is refused rather than duplicated. If somebody told you two records
were the problem, that is not a state this system produces.

If it IS in the list, you are confirming rather than onboarding — steps 3 onward,
and worth running on a schedule rather than once.

## 2. Choose the label the picker will look for

If you intend label-based pickup, the label has to exist in the repository first.
`customer.repos.labels.list` reads the repository's label names exactly as the
eligibility setting reads them, so you can pick one that is really there rather
than one you believe is there.

**Branch on `configured` before reaching for names.** This read answers in two
shapes and only one carries labels:

- `configured: false` — and nothing else. There are no label names in this
  response. The app cannot read the repository: access was never installed, or
  it was removed. That is a blocker to name, not an empty list to work around,
  and looking for a `labels` field here finds none.
- `configured: true` with `labels` — the names, ready to choose from.

A label that does not exist is not an error anyone reports to you. It is a
repository that considers nothing eligible, forever.

## 3. Read the configuration, in the customer's terms

`customer.repos.config` answers what this repository picks and in what order.
The response wraps them in a `config` object — eight fields cross the wire, and
here is what each one means for you:

| field | what it decides |
|---|---|
| `eligibilityMode` | `labeled` — only issues carrying the label are candidates. `all-issues` — every open issue is a candidate |
| `eligibilityLabel` | the label required in `labeled` mode. **Not consulted at all in `all-issues` mode** |
| `excludeAssigned` | whether assigned issues are skipped — see the warning below |
| `orderStrategy` | the direction a tier is walked: oldest issue first, or newest first |
| `orderQueries` | your saved walk, or `null` |
| `effectiveOrderQueries` | what the picker ACTUALLY walks, derived |
| `autoMergeEnabled` | whether a run's pull request merges itself once it is green |
| `commitPlanDocument` | whether runs also commit the plan document into the repository |

Three of those mislead if read at face value:

- **`orderQueries: null` is the normal case, not a missing setting.** It means
  "use the mode's default walk". Nothing is wrong and nothing needs setting.
- **`effectiveOrderQueries` is derived, never stored.** It is `orderQueries` when
  you have set one, otherwise the default walk. Do not write it back as
  configuration — `orderQueries` is the field that is set.
- **`excludeAssigned` is not a record of anybody's decision.** It is
  auto-managed: the system can flip and restore it on its own as a recovery
  measure. Reading `false` and concluding "someone chose to include assigned
  issues" is wrong.

## 4. Change what needs changing — and know what you can read back

`customer.repos.update` accepts exactly three things: `paused`, `orderStrategy`,
and `maxConcurrentRunsThisRepo`.

`changed: false` in the response **is a success, not a refusal.** It means the
repository was already in the state you asked for. The intent is recorded either
way, which is what step 6 reads.

Then read it back rather than trusting the write — with one exception you need to
know about, because it is the one place read-back cannot help you:

| what you set | can you read it back? |
|---|---|
| `orderStrategy` | **Yes** — `customer.repos.config` publishes it |
| `paused` | **Yes** — `customer.repos.poller-status` publishes `repo.paused` |
| `maxConcurrentRunsThisRepo` | **No.** It is not on any customer read |

**The concurrency sub-cap is write-only from here, and that is deliberate** —
the configuration read publishes what the repository PICKS, and a concurrency
cap is about how much runs at once, not about what gets chosen. It is not a bug
and not an oversight; a customer read for it is a product decision that has been
deliberately deferred rather than forgotten.

So for that field, your confirmation is step 6: the settings audit records the
intent you sent. Two further things are true of it and easy to get wrong:

- `null` **clears** the sub-cap. That is a real write, not "leave it alone" —
  afterwards the account-level cap alone governs.
- A value **above** your account cap is **inert, not rejected**. The effective
  cap is the lower of the two, so setting a big number here raises nothing and
  you will get no complaint telling you so.

Notice what is missing from the write list: the eligibility label and mode are
**readable but not settable here**. If those need changing, change them where
they are set and come back to step 3 to confirm.

## 5. Confirm it is actually live — this is the point of the task

Two reads, and they answer different questions. Do both.

**`customer.repos.poller-status` — is it running at all?** Branch on
`status.kind`; it is a union over many causes, because "not polling" has several
different remedies. `repo.paused` and `billing.paused` are the raw flags
underneath, and `lastPoll` is the stored tick.

Green light needs `lastPoll` to have actually moved. A repository that has never
polled reports so, and that is a blocker rather than a slow start.

**`customer.repos.poller-diagnostics` — did it see anything?** A live read of
what the picker would do right now: how many issues carry the label, how many
would be picked, and which buckets hold the rest.

"Saw nothing" is the answer that needs care, because it has three different
meanings and they are distinguished by fields, not by the count:

- Branch on `configured` first, then on `mode`. **Two of the three shapes report
  no counts at all** — `configured: false`, and `mode: "all-issues"` — because
  neither can be measured in label buckets. No counts is not "nothing eligible".
- Check `truncated` before reading any count as a repository total. The counts
  cover one page.
- When `customOrdering` is true, the buckets describe the default walk rather
  than what this repository actually picks.

If it is polling and genuinely nothing is eligible, that is a green light with a
finding: the repository works, and no issue currently qualifies. Say both.

## 6. Show the trail

`customer.repos.settings-audit` lists this repository's recorded settings
intents, newest first — `actor`, `action`, `changed`, `detail`, `createdAt`. It
answers "who changed this, and when" without asking support, and it is where the
concurrency sub-cap from step 4 is visible at all.

`customer.audit.list` is the wider view: this account's authenticated API calls
with `operationId`, `method`, `path`, `status` and `outcome` (`ok`, `refused` or
`failed`, derived from the status). Use it when the question is "did my change
even arrive" rather than "what is the setting now".

## Where this ends

- **Green light** — polling, `lastPoll` moving, and you can say what the
  diagnostics saw, including "nothing eligible yet" when that is the truth.
- **Blocker, named** — not connected to this account at all (step 1, and not
  fixable here); the app cannot read the repository, so the label read answers
  `configured: false` (step 2); paused, and by which of the two pauses; never
  polled; an eligibility label the repository does not have; or a configuration
  that cannot be changed from here.

"The update returned 200" is neither of those.

## Credential

Every operation here is authenticated by your own account key — the one
`descant login` stores, or `$DESCANT_API_KEY`. One of them writes
(`customer.repos.update`); the rest only read.
