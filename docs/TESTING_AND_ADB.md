# Testing Meldframe with ADB

**Audience:** testers, contributors and advanced users  
**Status:** current development-build documentation

Meldframe has an ADB-facing shell-control bridge for deterministic acceptance testing. It exists so tests can exercise the same catalogue, launcher and window-command surfaces used by the UI instead of relying on coordinate taps.

This is a **testing surface**, not a public automation API or plugin interface. Its command set and wire details may change with the implementation.

## Availability and security boundary

The receiver is present in current builds but **ships disabled**. Enable it explicitly in:

`Settings > Privacy and security > For developers`

Turning the switch on is only an intent/availability gate. It does **not** grant ordinary Android applications control of Meldframe. The receiver also requires Android's `DUMP` permission, a `signature|privileged` permission available to `adb shell`/system contexts rather than an ordinary installed app.

Therefore:

- default install: receiver disabled;
- switch enabled: receiver can resolve, but Android permission enforcement still applies;
- ordinary third-party app: the Settings switch does not give it the required privileged permission;
- ADB/system test context: can use the bridge when it is enabled.

Do not describe the Settings switch itself as the security boundary. The component-disabled default and Android permission check are separate gates.

## What the bridge can currently exercise

The current receiver exposes these acceptance operations:

| Operation | Meaning |
| --- | --- |
| `apps` | List the application catalogue using Meldframe's encoded application identities. |
| `windows` | List current Meldframe windows and report them as `active`, `open` or `minimized`. |
| `launch` | Launch a catalogue entry through the composed Meldframe launcher. |
| `close` | Close matching Meldframe window instances through `WindowCommands`. |
| `minimize` | Minimize matching Meldframe window instances. |
| `restore` | Restore matching Meldframe window instances. |
| `activate` | Activate matching Meldframe window instances. |

The identity printed by `apps` is the identity accepted by operations such as `launch`; testers should not invent or infer an `AppId` from a display name.

## What a successful test proves

The bridge deliberately routes operations through the live shell surfaces. A successful `launch`, for example, is evidence that the composed launcher accepted the request; it is stronger than directly constructing an internal backend from a test harness.

It still does **not** prove:

- that pointer/touch hit targets are correct;
- that keyboard focus and accessibility behavior are correct;
- that a feature works on devices other than the device under test;
- that privileged Android task-control providers work;
- that the bridge is a stable end-user CLI/API.

Use UI/device acceptance tests alongside the bridge for those claims. See [Keyboard and accessibility](KEYBOARD_ACCESSIBILITY.md) and [Compatibility and verification](COMPATIBILITY.md).

## Current verification evidence

The implementation repository records WSA verification of catalogue listing, launching internal applications, window-state reporting and minimizing through the ADB bridge. It also records verification that a fresh install leaves the receiver disabled, enabling the developer switch makes it available to the ADB test context, and disabling the switch silences it again.

Treat this as **WSA-specific recorded evidence**, not a general Android/OEM certification.

## Troubleshooting

If an ADB command produces no Meldframe result, check in this order:

1. Meldframe is running. The receiver cannot drive a shell that has not published its live control surfaces.
2. The developer switch is enabled in Meldframe Settings.
3. The caller is actually an ADB/system context with the required permission; enabling the switch does not authorize an ordinary app.
4. For a window operation, use the encoded identity returned by the catalogue rather than a guessed display name.
5. For launch failures, inspect the launch outcome and capability/runtime diagnostics instead of assuming the bridge itself failed.

## Relationship to future automation work

This bridge should remain narrow. A future Browser Broker, Guest Protocol, plugin SDK or other automation surface has different compatibility and security requirements. The existence of this acceptance receiver is not evidence that those APIs are implemented.

For implementation details and exact current command encoding, the `Hyperdroid-recovery` source and tests remain authoritative.