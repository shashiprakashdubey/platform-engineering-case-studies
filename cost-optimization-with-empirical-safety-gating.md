# Cost Optimization with Empirical Safety Gating

## Problem

Cloud spend had grown faster than usage, with the usual suspects: idle non-production compute
running nights and weekends, an artifact registry accumulating every image ever built, and no
easy visibility into which projects/services drove cost. The mandate was to cut spend — but the
real risk in any cost-cutting exercise is **deleting something that turns out to be
load-bearing** and causing an outage that costs far more than it saved.

## Constraints

- **No serving impact.** Nothing that traffic depends on may be deleted or throttled.
- **Reversible and staged.** Roll out to dev → staging → prod, so a mistake surfaces cheaply
  before it reaches production.
- **Auditable.** Each action must be explainable ("we deleted X because Y proved it was unused").

## Approach

Three levers, each gated by an **empirical safety check** rather than a guess:

### 1. Idle-compute scheduling
Non-production VMs were scheduled off outside working hours (nights + weekends). Before enabling,
a checklist proved each VM was safe to stop: data on persistent (not local) disk, and the
service auto-starts on boot. Savings on the order of a working-week's worth of idle hours — with
no risk to anything that only runs during the day anyway.

### 2. Artifact-registry cleanup — the interesting part
Deleting old images sounds trivial and is quietly dangerous: **"untagged" does not mean
"unused."** A running service can pin an image by **digest**, so an untagged image may still be
actively served. A naive "delete untagged" policy would take down running workloads.

So the cleanup shipped with a **serving-safety validator**: before any deletion, it cross-checks
the proposed delete-set against the **digests of images that traffic-serving revisions are
currently pinned to**, and **fails loudly** if the delete-set intersects them. The policy was
also split conservative-for-prod vs. more-aggressive-for-dev, and rolled out **dry-run →
enforce**, dev first. The validator is what turned "probably fine" into "provably won't touch a
served image."

### 3. Cost visibility
Billing data was exported to a warehouse with a set of analysis views — daily/monthly cost per
project and service, week-over-week trends, and simple anomaly flags (a cost that jumps beyond
its recent norm). This made the *next* round of optimization data-driven instead of a hunt, and
turned "spend went up" into an alert rather than a month-end surprise.

## What made it non-trivial

- **The digest-vs-tag trap.** The entire safety of the registry cleanup hinged on validating
  against *pinned digests*, not tags. Getting that wrong is an outage; getting it right makes the
  cleanup boring.
- **Staging discipline.** Every lever went dev → staging → prod with a dry-run first, so the
  blast radius of any mistake was a dev environment, not production.

## Verification

- Registry cleanup: the validator ran in dry-run and reported the exact delete-set and any
  intersection with served digests (there must be none) before enforcement was enabled.
- Scheduling: confirmed services came back healthy on their next scheduled start, with data
  intact.
- Visibility: the cost views reconciled against the billing console totals.

## Outcome

- Meaningful, sustained spend reduction from idle-compute scheduling and registry hygiene.
- **No serving impact** — because deletions were gated on empirical proof, not assumption.
- Ongoing cost visibility, so optimization is now continuous and data-driven.

## Transferable lesson

Cost cutting is a *safety* exercise wearing a *savings* costume. The savings are easy; not
causing an outage is the hard part. Gate every destructive action on an **empirical check that
fails loud** ("prove this image isn't served before deleting it"), stage dev-first, and you can
cut aggressively without gambling on production.
