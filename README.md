# Platform Engineering Case Studies

Short, honest write-ups of infrastructure changes I've led — each one a real problem, the
constraints that made it non-trivial, the approach, and how I knew it worked. They're the
*reasoning* behind the patterns in
[iac-security-patterns](https://github.com/teamiumtree/iac-security-patterns) and the
architecture in
[gcp-pulumi-reference-architecture](https://github.com/teamiumtree/gcp-pulumi-reference-architecture).

> Generic and anonymized. No company, customer, or environment-specific identifiers — the
> transferable engineering, not any one org's internals. Figures are illustrative.

## The studies

| Study | The hard part |
|---|---|
| **[Migrating from SA keys to WIF](migrating-from-sa-keys-to-wif.md)** | Removing long-lived CI credentials without a flag day, across many repos and environments. |
| **[Consolidating authoritative IAM bindings, race-free](consolidating-authoritative-iam-bindings-race-free.md)** | Merging duplicate authoritative bindings for the same role without the consolidation itself stripping members mid-deploy. |
| **[Adopting click-ops resources into IaC](adopting-clickops-resources-into-iac.md)** | Bringing hand-created production resources under management without an outage or a turf war. |
| **[Zero-downtime Shielded VM / Secure Boot rollout](zero-downtime-shielded-vm-secure-boot-rollout.md)** | Enabling a security feature that requires a VM stop, on VMs that couldn't all stop at once. |
| **[Cost optimization with empirical safety gating](cost-optimization-with-empirical-safety-gating.md)** | Cutting cloud spend without deleting something that turns out to be load-bearing. |

## The through-line

If there's one theme across all of these, it's **making high-blast-radius changes boring** —
by finding the version of the change that can't cause an outage, proving it can't with a
`preview` or an empirical check, and encoding the invariant so it can't silently regress later.
