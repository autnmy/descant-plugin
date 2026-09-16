---
name: grant-and-revoke-access
description: >-
  Give something or someone access, take it away again, and replace a
  credential without an interval in which nothing valid is presented. Decide
  what the key is FOR before minting it, rotate rather than revoke-and-mint,
  and finish on the trail that says the change happened.
operations:
  - customer.keys.list
  - customer.keys.create
  - customer.keys.rotate
  - customer.keys.revoke
  - customer.invites.list
  - customer.invites.create
  - customer.invites.revoke
  - customer.audit.list
---

# Grant and revoke access

Three different things get called "access" here and they have three different
answers. Decide which one you are doing before you touch anything:

- **A program needs to reach the API** — that is a **key**, and steps 1 to 5
  are the whole lifecycle: decide, mint, rotate, revoke, account for it.
- **Someone who is not on the product yet should be** — that is an **invite**,
  step 6. It hands a new person their own account; it does not put them in
  yours.
- **Someone already on the product should see YOUR runs and repositories** —
  that is **tenant membership**, step 7. There is no operation for it on this
  surface today, and step 7 says plainly what a person does instead.

Two words are used exactly as the API uses them, because they are not the same
thing and the difference decides step 6. A **tenant** is the workspace a key
authenticates: its repositories, its runs, its keys, its audit trail. An
**account** is the person who signed in and created that key. Everything in
steps 1 to 5, and steps 8 and 9, is the tenant's. Step 6 is the account's.

## 1. Decide what the key is FOR, before you mint it

A key carries **scopes**, and a scope is not something anyone types: it is
derived from the operation — the family from the operation's id, read or write
from its method — so a write cannot be filed under `read` by a typo. A scope
reads:

> `<family>:<read|write>`, from the closed set the contract publishes.

Two consequences, and the second is the one people trip on.

**The set is closed.** You cannot invent a scope, and a grant spelled in a way
this build does not recognise admits nothing rather than something — an unknown
grant is not a grant. So a key is scoped by picking from what exists, and the
operations you intend to call are what tell you which ones.

**A key can never widen itself.** This is the entire security model of the key
surface, and the contract states it at the point of minting:

> The scopes the new key holds. Must be a subset of the minting key's own — a
> key cannot widen itself.

`keys:write` is itself a scope, so a key that cannot mint cannot mint a key that
can. Work out the narrowest set that covers what this key will actually do, and
mint that — not "the same as mine", which is how a read-only job ends up holding
a credential that can rewrite repository settings.

### Where the chain starts is a person, and that is not a gap

Nothing in the API can bootstrap a key wider than the one making the call, and
a key created before scopes existed was backfilled without `keys:write` — so it
can neither list nor mint keys. The first key that can mint keys is therefore
minted by an owner in the dashboard, on the settings page, ticking its scopes
there. If `customer.keys.create` refuses everything you ask for, this is
usually why: ask an owner to mint you a key from the dashboard with the scopes
you need, and mint downwards from it after that.

## 2. Mint the narrowest key that does the job

`customer.keys.create` takes a `label` (trimmed, 1 to 64 characters), the
`scopes` from step 1, and an optional `expiresAt` — omit it for a key that does
not expire, and a value must be in the future. It answers with the key's summary
and one field that appears nowhere else:

> The secret, shown exactly once. It is stored hashed and cannot be recovered; a
> caller that loses it mints a new key.

Read that as an instruction about the **next thirty seconds**, not a warning
about the future. Before you do anything else with the response:

- **A person's own machine** — `descant login` stores it in the machine's
  credential store, which is the documented place for it.
- **A service** — your own secret store, whatever that is. Not a repository,
  not a log line, not the transcript of whatever you are doing right now.
- **A one-off command** — `$DESCANT_API_KEY` is the only way to hand a key to a
  plain HTTP client, and it is a variable in a shell, not a home for a secret.

Afterwards a key is named by its `keyPrefix`, which is what shows up in a list
and what you quote when you talk about it:

> `dsc_` plus the first eight characters of the secret: how a key is named after
> it is minted.

**A lost secret is rotated, not recovered.** Nothing on this surface can hand it
back — not the list, not the audit trail. Step 3 is the recovery, because a
rotation replaces the secret without changing what the credential is called or
what it may do.

**Two refusals worth recognising here.** `conflict` means you asked for a scope
the minting key does not hold, and it names which one — that is step 1's rule
firing, not a fault. And the mint is **not idempotent**: a retry after a lost
response mints a *second* key rather than answering with the first. If you are
not sure whether a mint landed, do not retry blindly — run `customer.keys.list`,
look at the newest rows, and revoke the extra one if there are two.

## 3. Rotate — the order that leaves no gap

This is the step people get wrong under pressure, and the wrong order causes an
outage exactly as long as your deployment takes.

`customer.keys.rotate` is a **mint with a retirement**: it creates a successor
carrying the target's label and scopes and a fresh secret, and sets the target
to stop authenticating at the end of a grace window. Both keys are valid at the
same time, and that overlap is the whole point.

**Do it in this order.**

1. **Rotate.** Call `customer.keys.rotate` on the key you are replacing. Choose
   `graceSeconds` deliberately:

   > How long the retired key keeps authenticating once the successor exists, in
   > seconds. `0` retires it at once; the default is one hour; at most one day.

   Pick a window longer than your slowest deployment, and remember the ceiling
   is one day — a rotation you cannot finish inside a day is not one call.
   `0` is for a key you believe is compromised, where a gap is better than an
   extra minute of exposure; it is not the default for tidiness.

2. **Now do the thing between the two steps, and it is yours to do: put the new
   secret everywhere the old one is presented, and restart or redeploy whatever
   holds it.** The successor's secret is in the rotate response and nowhere
   else, under the same reveal-once rule as step 2. Nothing happens
   automatically here.

   **A rotation you do not finish is not a spare key — it is a scheduled
   outage.** The retirement is set by the call in step 1, not by anything you do
   afterwards, so abandoning the rotation leaves the old secret deployed and
   dying on a timer: everything presenting it starts failing when the window
   closes, with nobody having touched anything that day. A rotation you start,
   you finish. If you cannot, you have not undone it — the way back is to deploy
   the successor anyway.

   You are working against a deadline the response hands you. `retired.expiresAt`
   is:

   > When the retired key stops authenticating: the earlier of its own expiry and
   > the end of the grace window.

   **Read it rather than doing the arithmetic yourself.** If the key you rotated
   already had an expiry sooner than the window you asked for, your real window
   is shorter than the number you passed — an expiry is never extended by a
   rotation.

3. **Confirm nothing is still presenting the old key** (step 4), and only then
   revoke it if you want it gone before the window closes. Letting the window
   close on its own is equally correct and needs no call.

### If you lose the rotate response, do NOT call it again

A rotation is no more idempotent than a mint, and the damage is worse. A replay
mints a **second** successor — a live, fully scoped credential whose secret was
revealed once, to nobody — and it shortens nothing further, so the retirement
deadline you never read is still the one that counts. You would then be holding
two rotations, one orphan you cannot enumerate by any other means, and a clock
you cannot see.

Recover by reading instead of writing. `customer.keys.list` is newest first, so
the successors are at the top, carrying the target's label: anything minted at
the time of your call that you do not hold the secret for is the orphan, and
`customer.keys.revoke` on its `id` ends it. Then rotate once more, deliberately,
and keep that response this time.

### What breaks if you revoke first and deploy second

A revoke takes effect **at once**. From the moment it lands until your
deployment finishes, everything still presenting the old key is refused, and the
refusal is a `401` in which unknown, revoked and expired are one indistinguishable
answer — so it reads to whoever is paged as "the key is wrong", not "the key was
just revoked by us".

It is worse than an outage, because it is also an outage you cannot see. A call
refused before a credential is accepted has no tenant to attribute, so **it
leaves no audit row at all**. Revoke-then-deploy therefore destroys the very
evidence step 4 uses to tell you who was still calling. Rotate first; the
overlap is what keeps both the service and the trail alive.

## 4. Find out what is still presenting the old key

Two reads, coarse and precise, and you want the precise one.

**Coarse — `customer.keys.list`.** Every key the tenant has minted, newest
first, active and revoked alike: a revoke is a soft one, so a revoked key stays
in the list with `revokedAt` set. Each row carries `id`, `label`, `keyPrefix`,
`scopes`, `expiresAt`, `createdAt`, `lastUsedAt` and `revokedAt`. `lastUsedAt`
is the quick answer: if it has not moved since you deployed, nothing has
presented that key since.

Two things about this list to know before you conclude anything from it:

- **It is paged, and nothing filters it.** There is no `revoked` filter and no
  `scope` filter — you read `revokedAt` on the rows you already have. `items`
  is one page; follow `nextPageToken` until it is `null`, which it is on the
  last page rather than being absent.
- **`lastUsedAt` is a timestamp, not a caller.** It tells you *that* something
  presented the key, never *what*.

**Precise — `customer.audit.list` filtered to the key.** The `principal` filter
takes:

> A principal exactly as a row stores it — `<kind>:<id>`. Malformed input is
> refused rather than answered with an empty page, because on an audit read an
> empty page reads as `this credential did nothing`.

For a key that is `api-key:<id>`, using the same `id` the key list returns and
`customer.keys.revoke` takes in its path. Each row carries the `method`, the
`path`, the `scope` the operation required, the `status`, an `outcome` of `ok`,
`refused` or `failed`, the `requestId` and `createdAt` — so you learn not just
that the retired key is still in use but exactly what it is being used to do.
It does not say which machine presented it: the row names the credential and
the capability, never the caller's address.

**When the rows stop after your deployment, the rotation is finished.** That is
the signal to act on, and it is the reason step 3 puts this before the revoke
rather than after.

## 5. Revoke what is finished with

`customer.keys.revoke` takes the key's id and stops it authenticating at once.
It is a **soft** revoke: the row stays in the list so you can still see it, with
`revokedAt` set. The answer carries an `outcome`:

> `already-revoked` on a repeat: the key was not active, so nothing changed.

So repeating a revoke is safe in the way that matters — it tells you it changed
nothing rather than doing something new.

Two edges worth knowing before you call it:

- **A key id that belongs to another tenant and one that belongs to nothing are
  one indistinguishable `not_found`.** You cannot use this to learn whether
  somebody else's key exists, and a `not_found` on your own id means you have
  the wrong id, not that the key is gone.
- **A key may revoke itself, and that call is its last.** If the credential you
  are authenticating with is the one you are revoking, everything after it
  fails — including the audit read in step 9. Do that one last, deliberately,
  or from a different key.

## 6. Invites bring a NEW person onto the product

`customer.invites.*` is the product invite: a link that lets someone who does
not have an account sign up. It is not how you add a colleague to your
workspace — that is step 7 — and the two are easy to confuse because both are
called an invite.

**Which key you use decides whose budget you spend.** Invites belong to the
**account that created the key**, never to the tenant's current owner. Two keys
on one tenant are therefore not interchangeable here: each spends its own
creator's budget and lists its own creator's invites. A key with no creating
account on record resolves to no account at all and answers `not_found` — the
same answer as asking about somebody else's invite.

- **`customer.invites.list`** — the invites, each with an `id`, a `status`, a
  `createdAt` and a `redeemedAt` that is `null` while unredeemed; the `budget`;
  and `referralStatus`. The budget is one of two shapes, and you branch on
  `unlimited`: `true` with `remaining: null`, or `false` with a `remaining`
  count. There is no third state to handle. **It is metadata only** — the code
  is never here.
- **`customer.invites.create`** — mints one and reveals its code, with no
  fields to send. The response's `code` is:

  > THE CODE, and this is the ONLY response that carries it. Surface it, hand it
  > over, and do not persist it — nothing can return it again.

  Spending the last slot answers `invite_limit_reached` — a well-formed request
  a rule refused, which is why it is not the malformed-body code. The mint is
  not idempotent: a replay mints a second invite and spends a second slot.
- **`customer.invites.revoke`** — kills the link and returns the slot to the
  budget. A missing invite, an already-terminal one and another account's are
  one indistinguishable `not_found`.

**There is no resend, and no way to read an invite back.** So "re-invite" is
two calls, in this order: revoke the invite whose code was lost, which puts the
slot back, then create a new one and surface the new code. Minting a second one
first spends a slot you did not need to spend.

## 7. Membership — who can see your runs — is the dashboard today

Adding a person to your tenant, changing what they can do, and removing them
have **no operation on this surface**. Do not go looking for one and do not
reach for something that sounds close: the invites in step 6 will not do it,
and no member operation exists here to find.

**What a person does instead:** an owner opens the dashboard's team settings
page and invites by email; the same page lists the pending invitations with a
revoke beside each, and the members with a remove. Inviting is owner-gated, so
a member cannot do it. An address that already has a live invitation is
refused as already invited and an address that is already a member is refused
as already a member — so a re-invite here is also revoke-then-invite, as in
step 6.

Closing this gap is tracked by **#14920**, which is widening the customer API to
cover every action a customer can take. When the membership operations land,
this step is what gets replaced.

**The question underneath, which is the one actually worth asking before you
invite anyone:** a tenant is a trust unit. Every member of it sees everything
the product surfaces for that tenant's repositories — run history, issue and
pull request titles, failure contexts, and stored excerpts that can include
fragments of code — **regardless of that member's own GitHub access to those
repositories**. A member's role governs what they may change, not what they may
see. There is no per-repository visibility setting, and there is
no way to invite someone to one repository: **the unit of sharing is the whole
tenant.** If that is more than you meant to share, the answer is a separate
tenant, not a narrower invitation.

One thing you may hit while inviting: once a tenant has repositories connected,
accepting an invitation requires the person to have a linked GitHub identity
that reaches **every** connected repository, and acceptance fails closed if
not. The
message the invitee sees is deliberately generic — it never names the private
repositories a non-member cannot see — so if someone reports that they cannot
accept, check their access to the connected repositories rather than trusting
the wording of the refusal.

## 8. The refusal you will actually meet, and where the scope is written

A key without the scope an operation needs is refused **by the gate, before the
request reaches the operation**. How that reads depends on how you are calling:

- **Over HTTP, or anything built on it** — `403` with a generic body. It does
  **not** name the scope you were missing. There is nothing in the response to
  read, so do not go hunting in it.
- **Through the tool surface** — the call is refused locally, before anything is
  sent, with a sentence that *does* name the operation and the scope it wanted,
  and says explicitly that nothing was sent. A refusal that says nothing was
  sent is not an estate failure and retrying it unchanged fails identically.

Either way the meaning is the same and it is not a fault: **a credential's
scopes are fixed when it is minted**. You cannot widen your own key, so this
needs whoever holds a wider key to mint you one — step 1's rule, seen from the
other side.

**When the body did not name the scope, the audit row does.** A refusal that
got past authentication is recorded, and the row carries the `scope` the
operation required beside an `outcome` of `refused`. That is the reliable way
to answer "which scope was I missing" after the fact — as long as the key you
read the audit with holds `audit:read`, which a key refused for a different
scope may not. Read it with a key that does.

## 9. End on the trail, because an undoable change is one you can see

`customer.audit.list` is the last step of every access change, not an optional
extra. An access change nobody can see later is one nobody can undo
confidently.

One row per authenticated call against this tenant, newest first, paged the same
way the key list is. Each carries the `principal`, the `operationId`, `method`,
`path`, `scope`, `status`, `outcome` and `requestId`. Two properties make it
useful for this task specifically:

- **A person's action in the dashboard leaves a row too**, as `user:<userId>`
  rather than `api-key:<id>`. That matters precisely here: the first
  key-minting key is minted by an owner on the settings page (step 1), so a
  trail of credentials alone would be blind at the event that creates the
  authority every other row depends on.
- **Refusals are in it.** A `403` for a key outside its scope, and a `500` for a
  call that failed after it was admitted.

Three things it does not say, each of which has bitten someone:

- **A `401` is never in it.** A call refused before the credential was accepted
  has no tenant to attribute, so a revoked key being presented leaves nothing
  behind. This is step 3's argument for rotating rather than revoking first.
- **A `429` is never in it** either. The only one this surface answers is the
  pre-authentication bound on the source, which refuses before a credential is
  read.
- **A mint's row does not say which key it produced.** The row names the
  capability exercised and who exercised it — for `customer.keys.create` the
  path is the collection, with no id in it. Line it up with `createdAt` on the
  key in `customer.keys.list` to say which key that call made. A revoke or a
  rotate is different: the key's id is a segment of the path, so those rows do
  name their target.

And an **empty page is not an answer**. Calls with no tenant to attribute are
absent entirely, so "nothing is here" means nothing got far enough to belong to
this tenant — not that nobody tried.

## What NOT to do

- **Do not revoke a key and then deploy its replacement.** That is an outage for
  the length of the deployment, and it erases the audit evidence of who was
  still calling. Rotate, deploy inside the grace window, then revoke.
- **Do not mint a key with the scopes you happen to hold.** Attenuation only
  bounds what you *can* grant; deciding what this key is *for* is step 1 and
  nothing does it for you.
- **Do not retry a mint or a rotation whose response you lost.** Neither is
  idempotent — a retry mints a second key, and a retried rotation mints a second
  successor while the deadline you never read keeps running. List, then revoke
  the extra.
- **Do not start a rotation you are not going to finish.** The retirement is
  scheduled by the rotate call itself, so an abandoned rotation is an outage on
  a timer rather than a spare key left lying around.
- **Do not go looking for a lost secret.** It exists in one response and is
  stored hashed. Rotate.
- **Do not read `customer.invites.*` as a way to add a colleague to your
  workspace.** It creates an account for somebody new and spends your own
  account's budget. Membership is step 7.
- **Do not invite someone to "just one repository".** Membership is
  tenant-wide; there is no per-repository visibility to grant.
- **Do not treat a `not_found` as proof a key or invite is gone.** Somebody
  else's and nothing at all are the same answer by design.
- **Do not revoke the key you are working with until everything else is done.**
  That call is its last, including the audit read.

## Where this ends

Every ending here is a decision, not a call that succeeded:

1. **A new capability is needed** — you minted the narrowest key that covers it,
   the secret is in a credential store rather than a transcript, and you know
   its prefix.
2. **A secret is stale, leaked or lost** — you rotated it, deployed the
   successor inside the window `retired.expiresAt` names, and confirmed the old
   key's rows stopped. Revoking it now is optional; letting the window close is
   equally finished.
3. **A credential is finished with** — you revoked it, the row is still in the
   list with `revokedAt` set, and nothing is presenting it.
4. **Someone new should be on the product** — you minted an invite from the key
   whose account's budget you meant to spend, and handed over the code the one
   time it existed.
5. **Someone should see this tenant's work** — you accepted that membership is
   the whole tenant, and an owner did it from the dashboard's team settings
   page. There is no operation for it here yet (#14920).
6. **A call was refused for a scope** — you read the required scope off the
   audit row rather than the response body, and asked whoever holds a wider key
   to mint one. You did not retry it unchanged.

If you cannot tell which of those you are in because a read came back empty,
that is **unknown** rather than a verdict: an empty audit page means nothing was
attributable, and the key list is paged. Finish the pages before concluding
anything.

## Credential

Every operation here is authenticated by your own account key — the one
`descant login` stores, or `$DESCANT_API_KEY`. The reads need `keys:read`,
`invites:read` and `audit:read`; the mints, rotations and revokes need
`keys:write` and `invites:write`. A key holding only the read scopes can walk
steps 1, 4 and 9 and the list half of step 6, and change nothing — which is the
right key for finding out what is going on before deciding to change it.
