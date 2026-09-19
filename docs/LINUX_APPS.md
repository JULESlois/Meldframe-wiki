# Linux application discovery

**Audience:** users, testers and contributors  
**Status:** current implementation documentation; checked against `Hyperdroid-recovery` main on 2026-09-19

Meldframe can discover applications from freedesktop `.desktop` entries in a registered guest/container application directory, merge them into its common application catalogue, and list eligible entries in Start.

This is an **application-discovery and Start-integration milestone, not Linux GUI execution support**. A discovered Linux application can appear in Start, but the current build deliberately refuses to launch it because the Linux GUI execution/presentation backend is not implemented as a product feature yet.

## What is implemented

The current implementation includes:

- `.desktop` file scanning and parsing;
- a Linux application catalogue/source feeding the common `AppDescriptorRepository`;
- stable Linux identities of the form `AppId.Linux(container, desktopId)`;
- Start-menu listing for applications from explicitly registered containers;
- the same namespaced identity across Start, desktop and taskbar consumers;
- locale-aware application names;
- `Hidden` and `NoDisplay` handling;
- `OnlyShowIn` / `NotShowIn` evaluation using `Meldframe` as the desktop identity;
- parsing of `Exec` into an argument vector without passing it through a shell;
- persistent registration of a container ID and the path containing its application entries;
- rescanning registered paths when the shell starts, so the catalogue reflects applications added or removed in the guest.

The parser has been exercised against real WPS Office 12.1.2 and Cylheim 4.11.1 desktop entries in addition to automated fixtures/tests. On WSA, all six WPS entries from a registered test directory were verified as visible in Start using the names resolved for a `zh_CN` device.

## Important current limitations

Discovery does **not** currently imply any of the following:

- launching an arbitrary Linux GUI application from its Start entry;
- a production Wayland compositor/presentation backend;
- product XWayland support;
- automatic discovery of every Termux/PRoot/chroot distribution;
- direct scanning of an arbitrary PRoot guest filesystem through a runtime provider — the current scanner needs a path visible to the Android process;
- resolving Linux icon themes — discovered Linux applications currently use the generic managed placeholder rather than their `.desktop` `Icon` theme asset;
- a stable end-user package installer;
- desktop-file MIME/open-with integration across Android and Linux;
- remote-application registration/listing;
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

The registration itself persisted across an APK reinstall in the current WSA validation because Meldframe stored the registered path and rescanned it at startup. This does not guarantee that the referenced path or guest contents survive every Android uninstall/update/storage scenario.

`linux` reports registered containers, discovered entries and the argument vector each executable entry would use. `uninstall` removes the registration; it does not uninstall software from the guest Linux environment.

See [Testing Meldframe with ADB](TESTING_AND_ADB.md) for the security and availability boundaries of this test bridge.

## What appears in Start

An eligible application from a registered Linux container is now a normal Start catalogue item. This fixes an earlier implementation gap where `mf apps` could report Linux entries while Start silently dropped every `AppId.Linux` item.

The identity remains namespaced. For example, a `firefox.desktop` entry registered under `debian` and one registered under `alpine` remain two applications rather than collapsing into a single `firefox` item. Pinning, desktop and taskbar consumers therefore receive the same container-aware identity used by the catalogue.

Remote applications are intentionally different: there is currently no user-facing remote-host registration gesture behind them, so `AppId.Remote` entries are still excluded from Start. Linux Start integration is not evidence that remote application support has landed.

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

## Linux GUI research status

There is now stronger research evidence than the simple word “roadmap” suggests, but it still must not be confused with user support.

A 2026-09-19 M3 experiment ran the real Cylheim 4.11.1 Linux GUI application under a nested sway compositor in WSL, captured the rendered desktop, and independently decoded a complete frame from wayvnc's RFB socket. Meldframe could load the noVNC client and establish the WebSocket connection through `adb reverse`.

The last presentation link did not work: noVNC stayed blank. The same blank result reproduced in headless Edge outside Meldframe, so the evidence narrows that failure to the noVNC/neatvnc development harness rather than demonstrating a Meldframe WebView failure.

The correct conclusion is therefore narrow:

```text
real Linux GUI app executes        proven in research PoC
compositor renders it              proven in research PoC
rendered frame reaches RFB wire    proven in research PoC
noVNC paints that frame            not proven; current PoC fails here
Start entry launches Linux GUI     not implemented
per-window Wayland integration     not implemented
```

WSL was used for this experiment because the development WSA guest had no network access to install the required stack. That does not make WSL a supported Meldframe runtime provider, and the experiment does not establish equivalent behavior on Android OEM devices, Termux, PRoot or AVF.

See [Runtimes](RUNTIMES.md) for the capability-probe and presentation details.

## Where this fits in the roadmap

A useful way to read the current state is:

```text
.desktop discovery and identity       implemented
            ↓
Start/application catalogue listing   implemented + WSA verified
            ↓
whole-display GUI chain research      partial PoC; app → RFB frame proven
            ↓
Linux GUI execution backend           not yet implemented as product support
            ↓
Wayland/XWayland presentation         roadmap
```

Service-backed Linux applications such as current Code/code-server are a separate, already working path: the Linux side runs a service and Meldframe presents its UI through Android WebView. That should not be confused with native Linux GUI application windows.

See [Runtimes](RUNTIMES.md), [Code](CODE.md), [Features and current status](FEATURES.md), and [Known limitations](KNOWN_LIMITATIONS.md).
