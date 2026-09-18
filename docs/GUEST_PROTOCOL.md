# Meldframe Guest Protocol

**Audience:** users who want to understand Linux/remote integration, extension authors and contributors  
**Status:** architecture/roadmap documentation; not a claim that a stable Guest Protocol is implemented  
**Implementation status checked against `Hyperdroid-recovery` main on 2026-09-18**

The **Meldframe Guest Protocol** is the proposed stable boundary between the Android-side Meldframe host and execution environments that live outside the main app process: Termux helpers, Linux guests, future AVF virtual machines, SSH/remote machines, and similar providers.

It exists to prevent a long-term failure mode where every runtime acquires its own collection of intents, shell scripts, ports and special cases.

## What exists today

There is already real host↔external-runtime integration, but it is narrower than the proposed protocol:

- the current external Termux integration can probe command execution through Termux `RUN_COMMAND`;
- Terminal can use the current ttyd loopback path;
- Code can supervise a code-server service and present it through a local WebView/gateway path;
- runtime and service abstractions exist on the Meldframe side.

These are useful implementation evidence. They are **not** evidence that a versioned, transport-independent Guest Protocol is already available to plugins or arbitrary Linux guests.

Do not read this document as setup instructions for a current `meldframe-agent` binary. No stable public guest-agent package or protocol SDK is documented as current.

## Why another protocol is useful

A runtime provider answers questions such as:

```text
Which execution environments exist?
Are they healthy?
What can they do?
Which one can satisfy this application plan?
```

The Guest Protocol answers a different question:

> Once Meldframe is communicating with an external environment, what stable operations and events can cross that boundary?

Without that distinction, integrations tend to become technology-specific:

```text
Termux → Android intent + shell command
AVF    → custom vsock messages
SSH    → ad-hoc shell scripts
PRoot  → local helper protocol
```

The desired model is instead:

```text
Meldframe semantic operation
          ↓
Guest Protocol
          ↓
transport adapter
          ↓
Termux / PRoot / AVF / SSH / other guest
```

Transport and semantics should be separate. A local Unix socket, Android IPC, vsock or SSH channel may carry the protocol without redefining what `Exec`, `Health` or `OpenUrl` means.

## Proposed first protocol surface

The roadmap currently identifies a deliberately small initial surface:

| Area | Purpose | Current status |
| --- | --- | --- |
| Hello / negotiation | identify peer and negotiate protocol/features | proposed |
| Health | report whether the guest/runtime is usable | partially represented by current runtime/service health, protocol itself proposed |
| Exec / process lifecycle | execute commands and observe/cancel them | Termux-specific execution exists; generic protocol proposed |
| PTY | interactive terminal process and streams | current ttyd path validates Terminal UX, not the final generic PTY protocol |
| Service lifecycle | start/stop/inspect long-running services | service supervision exists host-side; guest protocol proposed |
| Application discovery | discover guest applications and metadata | roadmap |
| Open URL / file | ask the host or guest to open a resource | roadmap / portal-related |
| File grants | scoped cross-boundary file access | roadmap |
| Clipboard | host↔guest clipboard integration | roadmap |
| Notifications | publish guest notifications through Meldframe | roadmap |
| Endpoint publication | safely expose a guest Web/service endpoint | current Code path is evidence; generic protocol proposed |
| Shutdown | orderly guest/runtime shutdown | roadmap |

Later extensions may cover Wayland integration, richer GPU reporting, browser automation and audio. They should not be forced into the first protocol version merely because they are desirable eventually.

## Relationship to RuntimeProvider

The Guest Protocol does not replace `RuntimeProvider`.

A provider is the host-side integration that discovers and describes candidate runtimes. A guest agent is one possible implementation mechanism behind that provider.

```text
ApplicationCoordinator / planner
             ↓
       RuntimeProvider
             ↓
      runtime instance
             ↓
   Guest Protocol client
             ↓
      guest-side agent
```

A provider may also use a different mechanism when a full guest agent is unnecessary. The architecture should not require every runtime to pretend it is a VM.

## Relationship to portals

The Guest Protocol and the proposed [Portal Framework](PORTALS.md) solve different layers.

```text
Linux application
      ↓
xdg-desktop-portal / guest integration
      ↓
Guest Protocol transport
      ↓
Meldframe PortalBroker
      ↓
Android / shell desktop service
```

Portal APIs define desktop-service semantics and policy. The Guest Protocol can carry those requests across a runtime boundary.

This separation is important: a Linux guest must not receive unrestricted Android host APIs merely because it can communicate with Meldframe.

## Relationship to Wayland

Wayland is not the Guest Protocol.

Wayland primarily carries Linux GUI surface/window/input semantics. The Guest Protocol carries runtime and desktop-integration control semantics.

A future first-class Linux application may use both:

```text
Linux app
├─ Wayland → window/presentation integration
└─ Guest Protocol / portals → files, URLs, notifications, runtime and host services
```

Neither implies the other is already implemented.

## Candidate providers

The same semantic protocol should eventually be implementable by several backends:

- external Termux helper;
- a curated Meldframe Runtime companion;
- PRoot guest agent;
- AVF Linux guest;
- SSH/remote agent;
- potentially a WSL agent for development/interoperability scenarios.

These are architecture targets, not a current support matrix. See [Runtimes](RUNTIMES.md) and [Compatibility and verification](COMPATIBILITY.md) for current evidence.

## Isolation is part of planning

A common mistake would be to treat every guest that implements the protocol as equally trusted.

The roadmap explicitly separates runtime capability from isolation properties. Relevant classes include host process, Android sandbox, PRoot, container, VM and remote execution. These are not a simple strongest-to-weakest ranking; they protect different boundaries.

In particular, **PRoot is a compatibility mechanism, not a strong security boundary**. A successful Guest Protocol handshake must therefore never be interpreted as proof that untrusted code is safely isolated.

## Protocol design requirements

Before this becomes a stable public API, the implementation should satisfy at least these constraints:

1. **Version negotiation.** Peers negotiate protocol and feature versions instead of assuming identical builds.
2. **Authenticated peer identity.** Caller identity should come from the transport/session where possible, not an arbitrary app-supplied string.
3. **Capability negotiation.** Missing operations are explicit and normal; no provider is required to implement everything.
4. **Request identity and cancellation.** Long-running exec/service operations need stable IDs and deterministic cancellation semantics.
5. **Structured errors.** Transport failure, unsupported operation, permission denial, runtime failure and process exit must not collapse into one generic error.
6. **Backpressure and stream framing.** PTY/stdout/stderr must remain correct for large and long-running streams.
7. **Scoped resource grants.** Files and host resources cross the boundary through grants, not unrestricted filesystem exposure.
8. **Reconnect semantics must be explicit.** A connection surviving, a process surviving and a UI session being reattachable are three different facts.
9. **Fail closed for privileged operations.** Unknown capability or lost authorization must not silently fall back to a more privileged path.
10. **Transport independence.** Semantic messages must not encode assumptions that only make sense for Termux intents, SSH, or vsock.

## What users should expect now

For the current development build, use the documented Termux, Terminal and Code workflows rather than looking for a generic guest-agent setup screen.

If those paths fail, diagnose the concrete layer that exists today:

```text
Termux package / RUN_COMMAND
        ↓
runtime probe
        ↓
service or ttyd/code-server
        ↓
loopback/gateway
        ↓
WebView presentation
```

See [Installation](INSTALLATION.md), [Terminal](TERMINAL.md), [Code](CODE.md) and [Troubleshooting](TROUBLESHOOTING.md).

## Acceptance criteria for a future first version

A first protocol should be considered useful only after at least two materially different providers can implement the same semantics without provider-specific changes in clients.

A strong validation pair would be:

```text
External Termux helper
        +
AVF or SSH/remote guest
```

with the same host client exercising negotiation, health, exec/process lifecycle, cancellation, endpoint publication and at least one scoped file/open operation.

Until that happens, the protocol should remain deliberately small and explicitly unstable.
