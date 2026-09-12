# Consolidating Authoritative IAM Bindings, Race-Free

## Problem

A foundation stack had ended up with **two authoritative `IAMBinding` resources for the same
role** (the result of two features each declaring the role's members independently). Because
`IAMBinding` is authoritative — it overwrites the role's *entire* member list on every apply —
the two bindings were **clobbering each other**: whichever deployed last won, and the other's
members silently disappeared on the next unrelated deploy. Intermittent "who removed their
access?" reports with no obvious cause.

The fix is to collapse them into **one** authoritative binding whose member list is the union.
The trap is that the *consolidation itself* can cause the very outage it's meant to prevent.

## Constraints

- **No access flapping.** During the change, no principal that legitimately holds the role may
  lose it, even for one deploy cycle.
- **Applies to a production project.** IAM changes here are the highest-scrutiny changes there
  are; the plan had to be provably safe *before* apply.

## The race

The naive fix — edit both bindings to the union in one pass and `pulumi up` — races. Pulumi may
apply the *delete* of the retired binding and the *update* of the survivor in an order where,
for a moment, the role's membership is empty or partial. On an authoritative binding, "for a
moment empty" means real principals lose access mid-deploy.

## Approach

Decouple the state operation from the resource change:

1. **Compute the union** of both bindings' members and make it the survivor binding's complete
   member list — sourced from config, not literals.
2. **`pulumi state delete` the retired binding's URN first.** This drops it from Pulumi's state
   *without touching the live role* — Pulumi simply stops tracking that resource; GCP's actual
   IAM is unchanged.
3. **`pulumi up --refresh` the survivor**, whose member list is now the full union. Refresh
   reconciles against live state so the apply is a no-op-or-additive convergence, not a
   delete-then-recreate.

This sequence means there is **never** a moment where an authoritative binding applies an empty
or partial member set to the live role.

## Making it not recur

The reason two bindings existed is that nothing *stopped* a second one from being added. So the
consolidation shipped with an **AST regression test** that parses the source and fails CI if any
role ever has more than one authoritative binding again. (That test is in
[iac-security-patterns/authoritative-iam](https://github.com/shashiprakashdubey/iac-security-patterns/tree/main/authoritative-iam).)
A comment saying "don't add another binding" would have rotted; a test doesn't.

## Verification

- **Pre/post member diff.** I captured the flattened set of `(role, member)` pairs before and
  after, and gated on `net_lost == 0` — no principal lost any role. This is a cheap, decisive
  "did anything break" check for IAM changes.
- **Clean `preview`.** The survivor's plan showed additive convergence only — no replace, no
  empty-set apply.

## Outcome

- One authoritative binding per role, holding the correct union of members.
- No access lost at any point during the migration.
- The class of bug is now **impossible to reintroduce silently** — CI enforces the invariant.

## Transferable lesson

When a resource is *authoritative*, consolidation is a state problem before it's a config
problem. Separate "stop tracking the old thing" (`state delete`) from "converge the survivor"
(`up --refresh`), and prove safety with a before/after diff — don't let one `up` try to do both
and race itself into an outage.
