# Runtime isolation and trust

**Audience:** users, extension authors and contributors  
**Status:** current security guidance plus roadmap boundaries; checked against `Hyperdroid-recovery` main on 2026-09-18

Meldframe can execute work through environments with very different security properties. A runtime being able to run Linux commands does **not** imply that it safely isolates untrusted code.

The implementation repository remains authoritative for the providers and enforcement mechanisms that actually exist. This page explains how to interpret them without turning roadmap security ideas into promises.

## The short version

- The current verified external Linux path is Termux-based execution on WSA.
- Android application sandboxing, PRoot path translation, a root/chroot environment, a VM and a remote machine are different trust boundaries; they are not interchangeable labels for "Linux runtime".
- **PRoot is a compatibility mechanism, not a strong security sandbox.** It changes how a process sees paths and selected system interactions; it should not be treated as equivalent to a VM boundary.
- Root access increases what an integration can do and therefore increases the consequences of mistakes or compromise. Root is not a security feature.
- AVF is a concrete long-term runtime target because VM isolation can provide a materially different boundary, but Meldframe does not currently document an AVF runtime as an implemented user feature.
- The future execution planner is intended to consider isolation/trust properties alongside capability, health and performance. That policy is roadmap work, not a current guarantee that untrusted workloads are automatically sandboxed.

## Compatibility and isolation are separate questions

A runtime candidate needs at least two independent evaluations:

```text
Can this environment run the workload?
              +
What can the workload reach if it is malicious or compromised?
```

For example, PRoot may make a Linux userspace convenient to run without root. That says something useful about compatibility and deployment cost, but it does not establish a strong security boundary.

Conversely, a VM may provide a stronger process/kernel boundary while lacking a capability the workload needs. The planner should therefore never collapse these properties into one generic `runtimeQuality` score.

## Runtime families and their security meaning

| Environment | Security interpretation | Current Meldframe status |
| --- | --- | --- |
| External Termux application | Work executes through another Android application and the permissions/interfaces it exposes | **Implemented path; strongest recorded verification is WSA** |
| PRoot | Userspace compatibility/path-translation mechanism; **not a strong isolation boundary** | **Roadmap/general provider not currently documented as implemented** |
| Root/chroot | Can provide a conventional Linux filesystem/process environment, but code may execute with much greater host authority depending on design | **Candidate/roadmap** |
| Android app sandbox | Android UID/process isolation applies to code actually contained by that sandbox; cross-app bridges still need explicit authorization | **Platform mechanism; exact Meldframe use depends on provider** |
| AVF/crosvm VM | Distinct VM isolation model suitable for a future stronger local runtime tier | **Research/roadmap target, not a current user runtime** |
| Remote/SSH runtime | Work executes on another trust domain; local isolation may improve while data/network trust and remote-host policy become relevant | **Candidate/roadmap** |

These rows are deliberately not ranked from "worst" to "best". A remote machine, Android sandbox and local VM protect different resources and introduce different trust assumptions.

## Root and privileged Android integration

Meldframe's Android task-control research includes an opt-in Magisk adapter. That work must not be confused with Linux runtime isolation.

A privileged task provider can gain authority to observe or manipulate Android tasks. Granting that authority does not make a Linux workload safer, and Linux workloads should not inherit privileged Android access merely because the shell has it.

A useful design rule is:

```text
privileged desktop operation
        ↓
small, explicit privileged adapter

untrusted workload
        ↓
no ambient privileged handle
```

The current WSA evidence also has an important boundary: selected task-control command semantics were validated from shell, while the recorded Meldframe-UID Magisk path had not yet completed a successful privileged round trip. See [Android desktop integration](ANDROID_DESKTOP.md).

## Why scoped grants matter

A runtime often needs user files without needing the whole host filesystem. The intended architecture therefore favors explicit grants:

```text
user selects file/folder
        ↓
Meldframe FileGrant / bridge
        ↓
selected runtime receives only the required access
        ↓
grant has an explicit mode and lifetime
```

`FileGrant` and a general cross-runtime file bridge are architectural/roadmap concepts; this diagram is not a claim that the complete portal is already implemented.

The principle is still useful now: do not treat broad shared-storage mounting as the default solution to every host↔guest file problem.

## Guest Protocol security requirements

The proposed Meldframe Guest Protocol is intended to replace ad-hoc provider-specific integration over time. Before it can be treated as a security boundary, it needs more than message serialization.

At minimum, a production protocol should define:

- authenticated peer identity;
- protocol and feature negotiation;
- authorization for privileged operations;
- scoped file/resource grants;
- request identity and cancellation;
- structured errors rather than silent fallback;
- stream limits/backpressure;
- reconnect and stale-session semantics;
- fail-closed behavior when authorization or capability evidence is missing.

The current Termux intents, ttyd and service paths are implementation evidence for designing this boundary. They are **not** a stable public Guest Protocol or security API today. See [Guest Protocol](GUEST_PROTOCOL.md).

## Intended planner model

The implementation roadmap now calls for a structured isolation profile in runtime capability detection. A conceptual profile may distinguish environments such as:

```text
host process
Android sandbox
PRoot
container-like environment
VM
remote runtime
```

The labels alone are insufficient. The eventual planner needs facts such as who owns the kernel, what host files are reachable, whether host Android APIs are exposed, how networking is scoped, and which privileged bridges are available.

That planner policy is **planned architecture**. Current users should not assume Meldframe automatically moves unknown executables into a VM or otherwise makes arbitrary downloaded native code safe to run.

## Guidance for users today

Treat executables and scripts run through Meldframe with the same caution you would use when running them directly in the underlying environment. In particular, do not interpret "runs in PRoot", "runs in Termux", or "runs in Linux" as a malware sandbox claim.

If a workflow requires a strong isolation guarantee, verify the actual runtime and enforcement boundary rather than relying on the Meldframe application label. Current public documentation does not claim a production-grade untrusted-code sandbox.

## Guidance for extension/runtime authors

A provider should report what it can prove, not infer security from technology names. Avoid implications such as:

```text
rooted == capable and safe
PRoot == sandboxed
remote == isolated therefore trusted
VM == unrestricted host integration
```

Privileged helpers should expose narrow semantic operations instead of generic root shells where possible. Missing authorization, unknown capability state, or a failed identity check should fail closed rather than silently selecting a more privileged path.

## Related documents

- [Runtimes](RUNTIMES.md)
- [Capabilities](CAPABILITIES.md)
- [Known limitations](KNOWN_LIMITATIONS.md)
- [Compatibility and verification](COMPATIBILITY.md)
- [Android desktop integration](ANDROID_DESKTOP.md)
- [Guest Protocol](GUEST_PROTOCOL.md)
- [Portal Framework](PORTALS.md)
