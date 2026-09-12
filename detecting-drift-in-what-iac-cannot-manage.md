# Detecting Drift in What IaC Cannot Manage

## Problem

Infrastructure as code gives you a comfortable assumption: if it is not in the code, it is not
in the estate, and `preview` will tell you when reality disagrees.

That assumption has a hole, and the hole is exactly where the security-relevant settings live.
A provider models most of a resource's fields, not all of them. When a field is **unmodelled**,
two things are true at once, and the second is the dangerous one:

1. nothing manages it, which you would expect; and
2. **nothing reports drift in it either.**

On an imported resource, an unmodelled field is invisible to `preview`. So a console edit to,
say, a password policy or a client-verification setting is silent from every angle the IaC
offers — no diff, no warning, no drift report. There is no version of that check a plan could
perform, because the plan has never heard of the field.

The same shape appears in three other guises: a field in `ignore_changes` the tool never
reconciles; a control written by another repository or inherited from the organisation; and a
resource with no deploy pipeline, where the next manual command silently reverts a setting.

## Constraints

- **Read-only, enforced.** A drift check must not be able to change what it is watching.
- **No credentials in CI that CI should not hold**, and no long-lived keys.
- **It must be impossible for the check to look healthy while not running.** That is the
  failure the whole exercise is about.

## Approach

Each unmanaged control got a scheduled, out-of-band checker built from the same five parts.

### A recorded expectation

The checker compares live API state against a small recorded map of what each environment should
be. The coupling is the mechanism: *if the change was intended, update the recorded value in the
same change that made it.* Drift is reported in **both** directions — an unannounced tightening
is as much a surprise as a loosening.

### A positive control

Before an absent field is allowed to mean "not configured", the response must prove it was
readable at all. Otherwise *"the policy is off"* and *"I could not read the config"* produce
identical output — and for most settings, "off" is the boring expected value, so the check goes
quietly green on exactly the environment you can no longer audit.

The CLI form is sharper: several policy commands exit non-zero with **empty stdout** both when a
policy is unset and when you lack permission. Gate on the return code, never on emptiness.

### Its own least-privilege identity

A custom role with exactly the permissions needed, not a predefined viewer role that also reads
user records — so read-only is guaranteed by IAM rather than by the script staying disciplined.
Federated credentials, pinned to a single repository *and ref*: a binding scoped to the
repository alone can be assumed from any branch in it.

### A schedule, and a guard on the schedule

The part easiest to skip and most often fatal. A checker can be written, reviewed, merged and
referenced in the architecture doc while being invoked by **no workflow** — reading as coverage
from every angle except the one that matters. So a second guard asserts it is actually wired:
invoked as the sole statement of its step, no `continue-on-error`, and a distinct notification
for *"the check could not run"* as against *"drift found"*.

## Outcome

Settings that no plan could ever have surfaced are now checked daily, by identities that cannot
write, with a failure mode that is loud rather than silent.

## Transferable lesson

**`preview` is blind to fields your provider does not model, and its silence is
indistinguishable from agreement.** Enumerate what your IaC genuinely cannot see — unmodelled
fields, ignored properties, controls owned elsewhere — and give each one an out-of-band check
on a clock. Then check that the check runs.
