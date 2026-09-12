# Migrating from Service-Account Keys to Workload Identity Federation

## Problem

CI/CD deployed to GCP using long-lived **service-account JSON keys** stored as GitHub repo
secrets. Across several repositories and environments, that's a spread of standing credentials,
each one:

- valid indefinitely (no natural rotation),
- extractable by any workflow step or compromised dependency,
- scoped to whatever the deploy SA could do — typically "deploy to production."

A single leaked key is a full compromise of the deploy path. The goal was to **eliminate the
keys entirely**, not just rotate them.

## Constraints

- **No flag day.** Deploys had to keep working throughout; I couldn't take CI offline for all
  repos at once.
- **Per-environment isolation.** Dev, staging, and prod deploy identities must stay separate —
  the migration couldn't collapse them.
- **Reversible per step.** If one repo's cutover misbehaved, I needed to roll just that one back
  without touching the others.

## Approach

**Workload Identity Federation (WIF).** GitHub's OIDC token is exchanged directly for a
short-lived GCP token — no key stored anywhere. (The Pulumi shape is in
[iac-security-patterns/keyless-ci-wif](https://github.com/shashiprakashdubey/iac-security-patterns/tree/main/keyless-ci-wif).)

The rollout was **incremental, environment by environment**:

1. **Stand up the pool + provider** trusting GitHub's issuer, with the trust condition pinned to
   **immutable numeric claims** (`repository_owner_id`, `repository_id`) rather than the mutable
   repo/org name — so a rename or transfer can't spoof the trust.
2. **Grant impersonation additively** — a per-(SA, repo) `workloadIdentityUser` binding, so
   enabling one repo never disturbs another's access.
3. **Cut over dev first**, run real deploys, confirm the federated token path works end to end,
   and only then move staging, then prod.
4. **Delete the old keys** for each repo *after* its keyless path was proven — never before.

## What made it non-trivial

- **Immutable-claim trust.** The obvious `attribute.repository == "org/repo"` condition is a
  latent hijack risk because names are mutable. Keying on numeric ids closed that.
- **No mid-deploy blind spot.** Impersonation bindings were changed with
  `delete_before_replace=False` so there was never an instant where a repo had *no* binding and
  an in-flight pipeline would fail.
- **Ordering.** Keys came out only after the replacement was verified for that repo, so a
  failed cutover degraded to "still using the old key," never to "no way to deploy."

## Outcome

- **Zero standing deploy credentials.** There is no key left to leak or rotate.
- **Deploys never stopped** during the migration.
- **Stronger trust than before.** Auth is now scoped to a specific repo *and* workflow via
  signed, short-lived tokens — a strictly better posture than a shared key.

## Transferable lesson

The best way to secure a secret is to **not have it**. When you must migrate credentials,
cut over in dependency order, prove the new path per unit before removing the old one, and
make each step independently reversible — so the migration itself can never be the outage.
