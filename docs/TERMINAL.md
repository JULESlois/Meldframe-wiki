# Meldframe Terminal

**Audience:** users, testers and contributors  
**Status:** implemented PoC; several interaction features remain incomplete

Meldframe Terminal is a Meldframe-owned terminal application. xterm.js renders the terminal, while
Meldframe owns the window, session model, runtime selection and transport abstraction.

It is deliberately not “a ttyd website inside a window”.

## Current architecture

```
xterm.js
   ↓
TerminalSession
   ↓
TerminalTransport
   ├─ TtydTransport        implemented
   ├─ direct Termux        direction
   ├─ Runtime Agent        direction
   ├─ local PTY            direction
   └─ SSH                  direction
```

Terminal input/output is byte-oriented. The transport does not decode arbitrary stream boundaries as
UTF-8, because a multibyte character may be split across reads.

## What has actually been verified

The implementation repository records two levels of verification.

First, protocol/session logic has unit and integration coverage: terminal profiles, pane trees,
sessions, WebSocket framing/handshake, ttyd protocol handling and a real ttyd integration test when a
`ttyd` binary is available.

Second, the UI PoC was exercised on WSA on 2026-09-16 against ttyd 1.7.7. That run verified:

- opening Terminal from Start;
- prompt and output rendering;
- typed command execution;
- measured terminal resize (`stty size`) before and after maximising;
- OSC title updates reaching the Meldframe title bar;
- taskbar window lifecycle;
- common function/navigation/modifier escape sequences;
- Ctrl+C interrupt;
- a full-screen `top` session;
- SGR colour rendering.

This is useful evidence, but it is not equivalent to broad device certification.

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
- a direct Meldframe-owned PTY transport.

The domain model already separates windows, tabs, panes and sessions so these features can be added
without redefining process identity.

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

A session can therefore eventually outlive the pane that displays it. Detaching a tab or reconnecting
a session should not require pretending the process moved between windows.

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

Not all of those providers are implemented today. The model is intentionally ready for more than the
initial ttyd/Termux path.

## Why xterm.js

xterm.js handles mature terminal concerns such as VT rendering, selection, CJK width, emoji,
scrollback, mouse reporting and keyboard composition. Meldframe still has to solve the Android side:
IME behavior, desktop keyboard mapping, lifecycle, runtime/process ownership and window integration.

The renderer is therefore replaceable infrastructure below the Terminal application model.

## ttyd limitations

The current ttyd path is useful but has an important lifecycle limitation: if its WebSocket
connection disappears, ttyd normally terminates the attached process. In a WSA background test, the
Android side slept, the connection eventually closed and the shell died. Meldframe correctly showed
the session as disconnected, but there was nothing to reattach to.

Real reattachment requires a transport/runtime that keeps the process alive independently, for
example a runtime agent, native PTY broker, or an explicit persistent layer such as tmux.

## Shell integration direction

Meldframe can learn semantic state from standard OSC sequences instead of scraping terminal text:

- OSC 7: current working directory;
- OSC 133: command boundaries/status integration;
- OSC 8: hyperlinks.

That can later support operations such as opening the current directory in Explorer or Code and
navigating command history semantically.

## Related documentation

- [Installation and setup](INSTALLATION.md)
- [Using Meldframe](USAGE.md)
- [Runtime and capability model](CAPABILITIES.md)
- [Extensions and plugins](EXTENSIONS.md)
- [Troubleshooting](TROUBLESHOOTING.md)
