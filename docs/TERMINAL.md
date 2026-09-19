# Meldframe Terminal

**Audience:** users, testers and contributors  
**Status:** implemented PoC; ttyd path verified on WSA; several interaction features remain incomplete

Meldframe Terminal is a Meldframe-owned terminal application. xterm.js renders the terminal, while
Meldframe owns the window, session model, runtime selection and transport abstraction.

It is deliberately not “a ttyd website inside a window”.

## What you can use today

The current working end-to-end path is **Terminal → `TtydTransport` → ttyd → shell**. In the recorded
WSA verification, ttyd ran outside Android and was reached through `adb reverse`; that is a
development/verification topology, not evidence that every device has a ready-made local ttyd
provider.

Once a ttyd-backed profile/session is available, the verified UI path supports ordinary typed command
execution, terminal output and colour rendering, Ctrl+C, common navigation/function/modifier key
sequences, terminal resize when the Meldframe window changes size, and OSC title updates in the
Meldframe title bar. A full-screen `top` session was also exercised.

Do not infer more from that list than was tested. In particular, physical Meta-key behaviour, IME
composition, mouse/wheel input, and complete `vim`/`tmux` interaction have not been verified broadly.

## Current architecture

```
xterm.js
   ↓
TerminalSession
   ↓
TerminalTransport
   ├─ TtydTransport        implemented
   ├─ direct Termux        direction only
   ├─ Runtime Agent        direction only
   ├─ local PTY            direction only
   └─ SSH                  direction only
```

Terminal input/output is byte-oriented. The transport does not decode arbitrary stream boundaries as
UTF-8, because a multibyte character may be split across reads.

The entries labelled **direction only** are architecture/roadmap directions. Their presence in the
transport model is not an implementation or compatibility claim.

## What has actually been verified

The implementation repository records two levels of verification.

First, protocol/session logic has unit and integration coverage: terminal profiles, pane trees,
sessions, WebSocket framing/handshake, ttyd protocol handling and a real ttyd integration test when a
`ttyd` binary is available. That integration test is skipped when the binary is absent, so a green
test run does not by itself prove that the real-ttyd case ran.

Second, the UI PoC was exercised on WSA on 2026-09-16 against ttyd 1.7.7. That run verified:

- opening Terminal from Start;
- prompt and output rendering;
- typed command execution;
- measured terminal resize (`stty size`) before and after maximising (`44 144` → `51 176` in that run);
- OSC title updates reaching the Meldframe title bar;
- taskbar window lifecycle;
- common function/navigation/modifier escape sequences;
- Ctrl+C interrupt;
- a full-screen `top` session;
- SGR colour rendering.

This is useful evidence, but it is one WSA verification point rather than broad Android/OEM device
certification.

## Current lifecycle caveat

The ttyd path couples the interactive process to its WebSocket connection. A recorded WSA background
test let Android sleep long enough for the connection to close; ttyd then terminated the shell.
Meldframe correctly reported the session as disconnected, but there was no surviving process to
reattach to.

For work that must survive presentation loss, do not currently assume Meldframe Terminal itself
provides persistence. A persistent layer such as tmux can move process lifetime outside the terminal
connection, while a future runtime agent/native PTY broker could provide first-class reattachment.
Neither future transport should be documented as implemented yet.

## What is not finished

The following should not be described as complete today:

- the visible multi-tab strip;
- visible split-pane UI and pane manipulation;
- profile picker UI;
- reconnect/reattach after transport loss;
- physical Meta-key calibration;
- IME composition coverage;
- mouse-button and wheel coverage;
- complete `vim`/`tmux` interaction verification;
- a direct Meldframe-owned PTY transport;
- direct Termux, Runtime Agent and SSH terminal transports.

The domain model already separates windows, tabs, panes and sessions so these features can be added
without redefining process identity. That model is design groundwork, not proof that their UI or
transport implementations exist.

## Windows, tabs, panes and sessions

They are four different concepts:

```
TerminalWindow
  └─ TerminalTab
      └─ PaneTree
          └─ TerminalPane
              └─ TerminalSessionId

TerminalSession
  = runtime + process + transport + lifecycle
```

The model allows a session to eventually outlive the pane that displays it. Detaching a tab or
reconnecting a session therefore need not pretend that the process moved between windows. Current
ttyd lifecycle behaviour, however, does not yet provide that persistence.

## Profiles select runtimes

A Meldframe terminal profile is stronger than “run this shell command”. It identifies a runtime
target plus shell/configuration.

Conceptually:

```
Termux          runtime: termux/native       shell: fish
Debian          runtime: proot/debian        shell: /bin/bash
Ubuntu Root     runtime: chroot/ubuntu       shell: zsh
Workstation     runtime: ssh/workstation     shell: zsh
```

These are examples of the profile model, not a list of currently supported Terminal providers. The
current verified terminal transport is ttyd. Consult [Runtime guides](RUNTIMES.md) and
[Compatibility](COMPATIBILITY.md) before treating a runtime example as available product behaviour.

## Why xterm.js

xterm.js supplies the terminal renderer and mature VT-facing machinery. That does not automatically
make every xterm.js capability work through Meldframe on Android: input has to survive Android IME and
keyboard handling, Meldframe's bridge, the transport, and the remote PTY.

This distinction matters for features such as mouse reporting, composition and keyboard shortcuts.
They remain Meldframe integration questions even when xterm.js supports the underlying terminal
feature.

## Shell integration direction

Meldframe can eventually learn semantic state from standard OSC sequences instead of scraping
terminal text:

- OSC 7: current working directory;
- OSC 133: command boundaries/status integration;
- OSC 8: hyperlinks.

The verified OSC behaviour today is title propagation used by the window title. The richer shell
integration above is a direction, not a completed feature. It could later support operations such as
opening the current directory in Explorer or Code and navigating command history semantically.

## When reporting a Terminal problem

Separate the failure by layer before assuming the renderer is at fault:

1. **Runtime/process:** is the shell/ttyd process actually alive?
2. **Transport:** did the WebSocket connect, disconnect or fail framing/protocol negotiation?
3. **PTY semantics:** does the process see the expected size and input bytes?
4. **Presentation/input:** is xterm.js rendering output, and did Android/WebView deliver the intended
   keyboard, IME or pointer event?
5. **Meldframe window lifecycle:** did minimise, backgrounding, sleep or close change the connection?

For a useful report, include the runtime/topology, ttyd version if applicable, Android/WSA environment,
whether the problem survives a reconnect/new session, and the smallest command or key sequence that
reproduces it. See [Troubleshooting](TROUBLESHOOTING.md) for the wider diagnostic model.

## Related documentation

- [Installation and setup](INSTALLATION.md)
- [Using Meldframe](USAGE.md)
- [Runtime guides](RUNTIMES.md)
- [Runtime and capability model](CAPABILITIES.md)
- [Compatibility](COMPATIBILITY.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Troubleshooting](TROUBLESHOOTING.md)
