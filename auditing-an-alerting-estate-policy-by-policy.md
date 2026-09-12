# Auditing an Alerting Estate, Policy by Policy

## Problem

An alerting estate accumulates. Alerts get added during incidents, copied between services, and
inherited from whoever set the platform up. Nobody removes any, because removing an alert feels
like removing coverage.

The result looks healthy — a long list of policies, most of them green — and behaves badly in
three ways. Some page for things nobody acts on at 3am, so the rotation learns to acknowledge
and go back to sleep. Some are labelled critical but carry only a chat channel, so they wake
nobody. And some **cannot fire at all**: their filter matches a log line the application stopped
emitting, or a metric that was never built.

That last category is the reason this is worth doing properly. An alert that cannot fire is
worse than no alert, because it *reads as coverage* — it appears in the inventory, it satisfies
an audit, and the gap it was supposed to cover is now invisible.

## Constraints

- **No blind deletion.** Removing an alert that turns out to matter is the failure mode of the
  exercise itself.
- **Every decision must be evidenced**, not asserted. "This looks noisy" is not a reason.
- **The result has to survive me.** A one-off cleanup regresses within two quarters unless the
  decisions are encoded.

## Approach

Every policy got classified against three independent questions, each answerable from data
rather than opinion.

### 1. Can it fire?

Check the backing signal for data over a meaningful window. Distinguish *"the metric exists and
has recorded nothing"* from *"the metric was never built"* — they look identical in a dashboard
and have completely different fixes. A policy that cannot fire is deleted, not demoted; leaving
it is what created the illusion.

### 2. Should it page, or chat?

The test is not severity, it is **action**: is there something a human must do *now*? If the
answer is "look at it in the morning", it is a warning. Demotions were ratified one at a time
and recorded, so a later reader sees the decision rather than guessing at it.

### 3. If it pages, can it actually reach someone?

Read the delivery path, not the label. A policy labelled critical carrying only a chat channel
looks correct on every individual line — the defect lives in the gap between the label and the
channel list.

### Then encode each answer

Each conclusion became a check rather than a note: every paging alert carries a runbook;
severity derives from one argument so label and routing cannot disagree; an alert with no
delivery target fails CI. Deliberate gaps went into a ledger with an owner and a review date, so
an exemption cannot quietly become permanent.

## What made it non-trivial

The audit has to read **live** state, not source. Every genuinely unrouted policy was created
outside the code path — by hand, by an older version, by another team — so a source-level test
would have reported a clean estate with complete confidence.

And a subtlety that inverts the usual instinct: **zero results must be a failure.** "No
offenders found" and "could not enumerate anything" produce the same empty list, and the second
one is what a broken audit looks like forever after.

## Outcome

A smaller estate in which every paging alert has a reason, a runbook, and a delivery path that
was checked rather than assumed — plus CI guards that make the next regression fail the build
instead of failing quietly.

## Transferable lesson

**An alert that cannot fire is a liability, not a neutral.** It converts an honest gap into a
false assurance, and false assurance is the expensive kind. When auditing alerting, the first
question is not "is this too noisy" — it is "could this ever fire, and if it did, would anyone
hear it?"
