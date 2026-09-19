---
name: onboard-a-repository-and-confirm-it-is-working
description: >-
  Take a repository from nothing to verifiably picking up work — connect it,
  activate it, read the configuration back in plain terms, change what needs
  changing, and confirm the poller is actually running before you call it done.
operations:
  - customer.repos.connect
  - customer.repos.activate
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
This Skill covers the whole path, starting from nothing: `customer.repos.connect`
attaches the repository, `customer.repos.activate` finishes onboarding it, and
everything after that — configure, verify, prove it is live — is here too.

What it still cannot do for you is INSTALL the Descant Apps. Connecting binds a
repository Descant's Apps can already reach; it does not grant that reach. If
neither App is installed on the owner account there is nothing to bind and the
connect is refused, so install first and connect second.

**Connecting does not mean running.** A connected repository lands `pending` and
paused-or-not independently, and both `connect` and `activate` answer with an
`awaiting` list naming what is still owed. Empty means nothing is owed. That list
is the honest answer to "is this working yet"; the status alone is not.

Finish on one of two answers — **a green light**, meaning the poller ran and you
know what it saw, or **a named blocker**. Never on "the call succeeded".

## 1. Find the repository, and get its id

`customer.repos.list` gives every repository this account holds, each with the
`id` every later step needs. Match on `owner` and `repo`.

**If it is not in the list, connect it** with `customer.repos.connect`, giving
the `owner` and `repo`. That answers with the same repository shape the list
returns, including the `id` every later step needs, plus the `awaiting` list.
Then `customer.repos.activate` on that id.

**If the connect is refused, that IS your blocker and this API cannot clear it**
— the refusal names what is wrong: neither App reaches the repository, another
account already holds it, or the key has no creating account to attribute the
binding to (mint a fresh key and retry).

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

`customer.repos.update` accepts exactly five things: `paused`, `orderStrategy`,
`maxConcurrentRunsThisRepo`, `eligibilityLabel` and `eligibilityMode`. Send only
the ones you mean to change; anything you leave out stays as it is.

`changed: false` in the response **is a success, not a refusal.** It means the
repository was already in the state you asked for. The intent is recorded either
way, which is what step 6 reads.

Then read it back rather than trusting the write — with one exception you need to
know about, because it is the one place read-back cannot help you:

| what you set | can you read it back? |
|---|---|
| `orderStrategy` | **Yes** — `customer.repos.config` publishes it |
| `eligibilityLabel` | **Yes** — `customer.repos.config` publishes it |
| `eligibilityMode` | **Yes** — `customer.repos.config` publishes it |
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

**Setting `eligibilityLabel` does not create the label.** The update stores the
name and answers `changed: true` whether or not the repository has that label.
If it does not, nothing becomes eligible and nothing tells you so — the step 2
hazard, reached from the other side. Pick the name from
`customer.repos.labels.list` first, and set only a label that is really there.

Two refusals come back as `invalid_body` rather than a stored value: a label that
could never match (empty, a `P0`-style priority name, one containing a comma, or
`blocked` / `waiting`), and a label spelled the same as one your label mapping
already uses. Switching `eligibilityMode` to `all-issues` keeps the stored label,
so switching back to `labeled` restores it.

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
- **Blocker, named** — a connect that was refused, and for which of its stated
  reasons (step 1); the app cannot read the repository, so the label read answers
  `configured: false` (step 2); paused, and by which of the two pauses; never
  polled; an eligibility label the repository does not have; or a configuration
  that cannot be changed from here.

"The update returned 200" is neither of those.

## Credential

Every operation here is authenticated by your own account key — the one
`descant login` stores, or `$DESCANT_API_KEY`. Three of them write
(`customer.repos.connect`, `customer.repos.activate`, `customer.repos.update`);
the rest only read.

`connect` additionally needs the key to have a CREATING ACCOUNT, because the
binding is attributed to it. A legacy key, or one whose creator was erased, is
refused — and `customer.keys.list` does not publish that field, so you cannot
check in advance. If you are refused for that reason, mint a fresh key.
