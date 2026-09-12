# Disaster Recovery with Infrastructure as Code

## Problem

"What happens if we lose it?" needed a real answer for every part of the estate — not a vague
"we have Terraform/Pulumi so we're fine." A cloud platform mixes **stateless** resources (safe
to recreate) and **stateful** ones (recreating them = data loss), and a DR plan that treats them
the same is a plan that loses data.

## Constraints

- **No data loss** for stateful resources, ever — that's the whole point of DR.
- **Deterministic, rehearsed recovery** — the procedure must be written down and practiced, not
  invented under pressure during an incident.
- **The recovery artifacts must live off the affected infrastructure** — a backup that dies with
  the thing it backs up is not a backup.

## Approach

**Split the world by whether it holds data, and give each half its own recovery story.**

### Stateless → rebuild from code
Networks, load balancers, Cloud Run services, IAM, alert policies hold no data and are fully
described by Pulumi. Recovery is `pulumi up` in dependency order (`foundation` first). The only
artifacts needed are the **git repo + the Pulumi state**, both stored independently.

### Stateful → protect, then snapshot-recover
Data-bearing resources get three layers:

- **`deletion_protection` / `protect=True`** — can't be deleted by accident.
- **`retain_on_delete=True`** — survives even being removed from a stack.
- **Point-in-time recovery** — bucket **versioning + soft-delete** and disk **snapshots** give a
  known-good rollback for corruption, a bad write, or ransomware.

### The tricky operation: recovering a stateful disk
Some disk changes can't happen in place, and a naive `pulumi up` would *replace* the disk and
destroy the data. The data-preserving procedure:

```
snapshot the disk → create the new disk FROM the snapshot → pulumi refresh
   (ignore_changes on snapshot/description so the reconcile doesn't churn)
```

A "replace = data loss" operation becomes snapshot → recreate → reconcile, with a rollback point
at every step.

### Back up the state itself
Pulumi state is the map of the infrastructure, so it's part of the DR plan:
`pulumi stack export` before any high-risk operation, `pulumi stack import` to restore a
corrupted checkpoint — rather than rebuilding the mapping by hand.

## What made it non-trivial

- **The stateless/stateful split is the whole game.** The failure mode is treating a stateful
  resource like a stateless one and cheerfully recreating it empty. The design makes the two
  categories explicit and protects the stateful ones by default.
- **Blast radius is per-stack.** Because the estate is many independent stacks, losing one
  doesn't force a full-estate rebuild — you recover just that stack.

## Verification

- **Rehearsal, not theory.** The dangerous procedures — disk snapshot-recovery, state
  export/import — were actually exercised, so the runbook is proven, not aspirational.
- Post-recovery gate: `pulumi preview` clean, `stack output` resolves, smoke tests pass.

## Outcome

- Every resource class has an explicit, tested recovery path.
- Stateless recovery is a `pulumi up`; stateful recovery is a snapshot restore + `refresh`.
- The recovery artifacts (repo, state, snapshots, bucket versions) all live independently of the
  infrastructure they protect.

## Transferable lesson

DR isn't "do you have IaC" — it's "do you know, per resource, whether recovery means *recreate*
or *restore*, and have you rehearsed the restore." Split stateless from stateful, protect the
stateful by default, keep your backups (including the IaC state) off-box, and practice the scary
operations before you need them.

*(Mechanics reference:
[gcp-pulumi-reference-architecture › disaster-recovery](https://github.com/shashiprakashdubey/gcp-pulumi-reference-architecture/blob/main/docs/disaster-recovery.md).)*
