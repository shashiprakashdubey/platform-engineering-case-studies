# Adopting Click-Ops Resources into IaC

## Problem

Several production resources existed **outside** the IaC — a VM created from a Marketplace
image, a Cloud NAT stood up by hand during an incident, and provider-managed load-balancer
proxies. They worked, but nothing enforced their security posture, and every `pulumi up` on the
surrounding stacks risked fighting them. Two failure modes were live at once:

- **Security drift** — a hand-created VM had deletion protection off and Shielded VM disabled,
  and nothing would flag it if that regressed further.
- **Config drift** — an emergency Cloud NAT hotfix (dynamic port allocation to stop source-port
  exhaustion) lived only in the console, so the "source of truth" IaC didn't reflect reality.

## Constraints

- **No outage, no data loss.** These are production; delete-and-recreate-from-code was off the
  table — especially for the stateful VM.
- **Minimal ownership.** I wanted to *enforce a guarantee* on each resource, not become the
  authority for every field the Marketplace/operator lifecycle legitimately manages.

## Approach

**Adopt, don't rebuild.** Bring each resource under Pulumi *in place* via `import`, then manage
only the one or two invariants that matter and ignore the rest. (The Pulumi shape is in
[iac-security-patterns/adopting-clickops-into-iac](https://github.com/teamiumtree/iac-security-patterns/tree/main/adopting-clickops-into-iac).)

Per resource:

- `opts.import_` to adopt the existing object rather than create a new one.
- `protect=True` so an accidental `destroy` can't take out a production resource.
- `retain_on_delete=True` so if the resource ever leaves the stack, GCP keeps it.
- A tight set of **managed fields** (the invariants), and everything else in `ignore_changes`.

For the VM: enforce `deletion_protection=True` and Shielded VM (Secure Boot + vTPM + Integrity
Monitoring); ignore machine type, disks, NICs, metadata, service account. For the Cloud NAT:
enforce dynamic port allocation and `ERRORS_ONLY` logging; leave the parent router referenced
by name and unmanaged.

## What made it non-trivial

- **Import must adopt in place, not replace.** Every managed field had to be set to *match live
  state exactly*, so the first `preview` showed a single `[import]` with **zero updates**. A
  mismatch would have queued a replacement — the outage I was avoiding.
- **EIM vs dynamic ports on Cloud NAT.** GCP forbids dynamic port allocation while endpoint-
  independent mapping is on. Dev (no exhaustion risk) kept EIM; staging/prod took dynamic ports.
  The adoption had to encode that per-environment difference through config, not code branches.

## Verification

- First `preview` per resource: exactly one `[import]`, zero updates — proof the adoption was
  in-place.
- Post-adoption, deliberately checked that flipping an invariant in the console (e.g. Secure
  Boot off) now shows up as a `preview` diff — proof the guarantee is actually enforced, not
  just recorded.

## Outcome

- Hand-created production resources are now **governed** — their security invariants can't
  silently regress, because a regression is a visible diff.
- No outage and no replacement during adoption.
- The emergency hotfix is now codified, so IaC and reality agree.

## Transferable lesson

You don't have to *own* a resource to *govern* it. Adopt with `import` + `protect` +
`retain_on_delete`, manage only the invariants you care about, and ignore everything else — the
smallest blast radius that still makes the guarantee real.
