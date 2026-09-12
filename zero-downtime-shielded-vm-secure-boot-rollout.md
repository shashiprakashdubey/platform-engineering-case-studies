# Zero-Downtime Shielded VM / Secure Boot Rollout

## Problem

A security audit flagged that several production VMs did not have **Shielded VM** fully enabled —
specifically **Secure Boot**, alongside vTPM and Integrity Monitoring. Enabling Secure Boot
hardens a VM against boot-level and kernel-level tampering, and it was a required remediation.

The catch: **enabling Secure Boot requires the VM to be stopped**. It's not a live-patchable
setting. So a security requirement collided with an availability requirement.

## Constraints

- **Some VMs back user-facing services** — they can't all be down at once.
- **One VM was a remote-access appliance** — mishandling its stop/start could lock operators out
  of the very network they'd need to fix it. That one was a genuine landmine.
- **The change had to stick** — the audit finding must not silently reopen later.

## Approach

Treat it as a **staged, one-VM-at-a-time** rollout with an explicit safety checklist per VM,
rather than a fleet-wide flip:

1. **Sequence by risk.** Non-critical VMs first to validate the exact stop → enable → start
   procedure and timings, before touching anything user-facing or the access appliance.
2. **Per-VM safety checklist** before each stop:
   - Confirm persistent disks survive the stop (data is on persistent, not local, disk).
   - Confirm the service auto-starts on boot (systemd unit enabled) so recovery is automatic.
   - For the access appliance: confirm an **independent** recovery path exists before touching
     it — never rely solely on the thing you're rebooting.
3. **Enforce the invariant in code.** Rather than flip Secure Boot by hand and hope it stays,
   the VMs were brought under IaC with Shielded VM (Secure Boot + vTPM + Integrity Monitoring)
   as a *managed invariant* — so a regression becomes a visible diff. (Adoption pattern:
   [iac-security-patterns/adopting-clickops-into-iac](https://github.com/shashiprakashdubey/iac-security-patterns/tree/main/adopting-clickops-into-iac).)

## What made it non-trivial

- **"Zero downtime" was per-service, not per-VM.** Individual VMs *did* stop; the design goal
  was that no *service* had a user-visible outage. Staging + auto-start made each stop a brief,
  absorbed blip rather than an incident.
- **The access-appliance landmine.** The realistic failure was locking myself out. The mitigation
  was ordering (do it last, after the procedure was well-understood) and an independent recovery
  path proven *before* the stop.

## Verification

- After each VM: confirmed Secure Boot / vTPM / Integrity Monitoring all report enabled, the
  service came back healthy, and data was intact.
- Post-rollout: a `preview` on the managing stack is clean — and flipping a flag off out-of-band
  now shows as a diff, so the audit finding can't silently reopen.

## Outcome

- All targeted VMs on full Shielded VM, with no user-facing outage.
- The invariant is now enforced by IaC, not by memory — the finding stays closed.

## Transferable lesson

When a security change requires downtime, don't fight the downtime — **shrink and stage** it.
One unit at a time, a safety checklist per unit, riskiest last, and an independent recovery path
before you touch anything you depend on. Then encode the result as an invariant so the fix
doesn't rot.
