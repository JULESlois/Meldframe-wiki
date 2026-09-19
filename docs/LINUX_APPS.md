# Linux application discovery

**Audience:** users, testers and contributors  
**Status:** current implementation documentation; checked against `Hyperdroid-recovery` main on 2026-09-19

Meldframe can now discover applications from freedesktop `.desktop` entries in a registered guest/container application directory and merge them into its common application catalogue.

This is an **application-discovery milestone, not Linux GUI execution support**. A discovered Linux application can appear in Start, but the current build deliberately refuses to launch it because the Linux GUI execution/presentation chain is not implemented yet.

## What is implemented

The current implementation includes:

- `.desktop` file scanning and parsing;
- a Linux application catalogue/source feeding the common `AppDescriptorRepository`;
- stable Linux identities of the form `AppId.Linux(container, desktopId)`;
- locale-aware application names;
- `Hidden` and `NoDisplay` handling;
- `OnlyShowIn` / `NotShowIn` evaluation using `Meldframe` as the desktop identity;
- parsing of `Exec` into an argument vector without passing it through a shell;
- persistent registration of a container ID and the path containing its application entries;
- rescanning registered paths when the shell starts, so the catalogue reflects applications added or removed in the guest.

The parser has also been exercised against real WPS Office 12.1.2 and Cylheim 4.11.1 desktop entries in addition to automated fixtures/tests.

## What is not implemented

Discovery does **not** currently imply any of the following:

- launching an arbitrary Linux GUI application;
- a Wayland compositor/presentation backend;
- XWayland support;
- automatic discovery of every Termux/PRoot/chroot distribution;
- a stable end-user package installer;
- desktop-file MIME/open-with integration across Android and Linux;
- a stable public Guest Protocol.

The missing execution backend is intentional. Meldframe keeps an application visible as a known catalogue item while refusing an execution plan it cannot actually satisfy, rather than pretending discovery proves launch capability.

## Registering entries for testing

The current registration surface is exposed through the opt-in ADB acceptance bridge and is intended for development/testing, not as the final user installation UX.

With that bridge enabled, the helper supports:

```text
mf.sh install debian /path/to/applications
mf.sh linux
mf.sh uninstall debian
```

`install` associates a container ID with either a `.desktop` file or a directory containing entries. The registration stores the **path**, not a snapshot of parsed applications. On a later shell start Meldframe rescans that path.

`linux` reports registered containers, discovered entries and the argument vector each executable entry would use. `uninstall` removes the registration; it does not uninstall software from the guest Linux environment.

See [Testing Meldframe with ADB](TESTING_AND_ADB.md) for the security and availability boundaries of this test bridge.

## Container identity matters

The same desktop ID in two containers represents two different applications to Meldframe. For example, `firefox.desktop` in `debian` and `firefox.desktop` in another guest do not collapse into one catalogue entry.

Container IDs must not contain `:` because that character participates in Meldframe's encoded application identity.

## Desktop-entry semantics

A `.desktop` file is not safely interchangeable with a generic INI parser. The implementation handles desktop-entry escaping, list escaping and field-code expansion explicitly.

Two visibility fields are also intentionally different:

- `Hidden=true` means the entry is treated as removed and is not installed into the catalogue;
- `NoDisplay=true` keeps the application known but omits it from normal listing surfaces.

For desktop-environment filters Meldframe identifies itself as `Meldframe`; it does not claim to be GNOME, KDE or another desktop merely to make their session-specific components visible.

## Exec is not a shell command

Meldframe parses the desktop entry's `Exec` value into arguments and applies supported field-code substitutions to that argument vector. It does **not** concatenate the result into a string and execute it through a shell.

That distinction is both semantic and security-relevant: a filename containing shell metacharacters must remain a filename rather than becoming another command. A field code that expands to nothing removes its argument rather than manufacturing an empty positional argument.

This parsing work establishes what an eventual Linux execution backend should run. It does not make that backend exist today.

## Where this fits in the roadmap

A useful way to read the current state is:

```text
.desktop discovery and identity       implemented
            ↓
common Start/application catalogue    implemented
            ↓
Linux GUI execution backend           not yet implemented
            ↓
Wayland/XWayland presentation         roadmap
```

Service-backed Linux applications such as current Code/code-server are a separate, already working path: the Linux side runs a service and Meldframe presents its UI through Android WebView. That should not be confused with native Linux GUI application windows.

See [Runtimes](RUNTIMES.md), [Code](CODE.md), [Features and current status](FEATURES.md), and [Known limitations](KNOWN_LIMITATIONS.md).