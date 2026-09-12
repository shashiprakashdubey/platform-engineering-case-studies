# Standing Up a New UAT Environment with Pulumi

## Problem

The platform had two environments — development and production — and needed a third: a **UAT /
staging tier** where release candidates could soak in a prod-faithful environment before going
live. The ask was a *complete* new environment: its own project, network, compute, monitoring,
alerting, and DNS — not a shared namespace bolted onto an existing tier.

## Constraints

- **Faithful to prod, cheaper than prod.** UAT had to mirror prod's *topology* (so testing
  there is meaningful) while running smaller/scaled-to-zero to keep the bill down.
- **Fully isolated.** UAT's IAM, quota, billing, and network must not be able to affect dev or
  prod.
- **No forking the codebase.** I did not want an `if env == "uat"` sprinkled through every stack,
  or a copy-pasted parallel set of programs to maintain forever.

## Approach

The infrastructure was already built as **many independent Pulumi stacks, configured per
environment** (`Pulumi.<env>.yaml`) and wired together with `StackReference`. That design made a
new environment a matter of **instantiating the existing stacks for `uat`** — no new code.

For each stack, in dependency order (`foundation` → `networking` → compute/monitoring/alerting):

1. `pulumi stack init …/uat` — a new stack *instance* of the same program.
2. Author `Pulumi.uat.yaml` — starting from the closest existing env and adjusting the values
   that actually differ.
3. `pulumi preview` — confirm an all-creates plan (a new environment should touch nothing else).
4. `pulumi up`.

Because every cross-stack reference is env-parameterized —
`StackReference(f"org/foundation/{pulumi.get_stack()}")` — the UAT stacks wired themselves
together automatically; nothing was hardcoded to prod.

## What actually took judgment

The commands were rote. The **values** were the engineering:

- **A dedicated `platform-uat` project** for hard isolation of IAM/quota/billing.
- **Non-overlapping subnet CIDRs** — the one value that can't be copied from another env, or
  future connectivity breaks.
- **Right-sizing** — prod's shape, smaller machine types, `min_instances: 0` where cold starts
  are acceptable, so UAT is cheap.
- **Isolated DNS + managed certs** under `*.uat.example.com`.
- **Deploy order = dependency graph** — deploy a consumer before its producer and its
  `StackReference` resolves against a stack that doesn't exist yet.

## Verification

- Each stack's **first `preview` for `uat` was all-creates, zero-updates** — proof the new
  environment was clean and wasn't reaching into dev or prod.
- `pulumi stack output` resolved on every UAT stack, and downstream stacks read *UAT* outputs,
  not prod's.
- A smoke test against `*.uat.example.com` confirmed it was actually serving.

## Outcome

- A complete, isolated UAT tier — its own project, network, compute, and observability — stood
  up **without writing a line of new stack code**.
- The environment is a faithful, cheaper rehearsal space for release candidates.
- Adding a *fourth* environment later is now the same known procedure.

## Transferable lesson

If adding an environment means editing code, the architecture is wrong. Make every stack
**config-driven and reference-parameterized**, and a new environment collapses to "instantiate
the stacks with a new values file, deploy in dependency order." The mechanics are boring on
purpose; the only real thinking is isolation, addressing, and sizing.

*(Mechanics reference:
[gcp-pulumi-reference-architecture › adding-an-environment](https://github.com/shashiprakashdubey/gcp-pulumi-reference-architecture/blob/main/docs/adding-an-environment.md).)*
