# Keyboard, pointer and accessibility behavior

**Audience:** users, testers and contributors  
**Status:** implemented in the current UI code; representative hardware/accessibility-service validation is still needed

Meldframe is a desktop-oriented Android shell, so mouse, keyboard and accessibility behavior are not optional polish. Recent UI work has made context menus and several action controls explicitly keyboard-focusable and accessibility-addressable. This page records what the implementation currently intends to provide without treating code review as device certification.

## Current evidence boundary

The implementation repository is the source of truth. Current code contains keyboard navigation for the shared `UIContextMenu`, keyboard-triggered context menus in Start, pointer-coordinate anchoring work for Start/taskbar menus, and explicit content descriptions/focusability for action controls such as installed-app and widget actions.

These are **implemented behaviors**. They should not yet be described as broadly verified accessibility support: the current repository history explicitly places this work before device validation, and the wiki does not have evidence of a representative TalkBack, hardware-keyboard and OEM test matrix.

## Context menus

The shared context-menu implementation currently supports these keyboard operations when focus is inside a menu:

| Key | Intended behavior |
| --- | --- |
| Up / Down | Move between enabled menu items, wrapping at the ends |
| Home | Focus the first enabled item |
| End | Focus the last enabled item |
| Right | Open a submenu when the focused item has one |
| Left | Close a submenu and return focus to its parent item |
| Escape | Dismiss the menu chain |

Disabled entries and separators are skipped by keyboard traversal.

When a Start-menu context menu is opened from a keyboard context-menu action, the implementation asks the first enabled menu item to receive focus. Pointer/long-click opening does not force that same focus transition.

## Pointer placement

Recent implementation work normalizes context-menu coordinates and anchors Start/taskbar menus to the pointer rather than treating every menu as if it originated from a fixed location. Submenus remain anchored to their parent row.

This is important on a desktop shell, but it is still sensitive to Android window/display coordinate behavior. Multi-display, display scaling and OEM desktop modes require real-device verification; do not infer broad correctness from the implementation alone.

## Accessibility labels

Several icon-only action affordances now expose explicit labels and are focusable. Examples in the current implementation include:

- installed-app `App actions`;
- widget `Widget actions`.

The purpose is to avoid an unlabeled ellipsis being the only representation exposed to keyboard or accessibility navigation.

This does **not** establish WCAG conformance, complete TalkBack coverage or a finished accessibility audit. Those require systematic testing of the whole UI rather than isolated semantics fixes.

## What testers should verify

For a useful validation pass, test at least:

1. hardware keyboard navigation through Start and taskbar context menus;
2. Context Menu/Menu key invocation where the keyboard exposes one;
3. submenu Right/Left navigation and focus restoration;
4. Escape dismissal;
5. disabled-item skipping;
6. mouse right-click and long-press placement near all screen edges;
7. menus on secondary displays and under non-default display scaling;
8. TalkBack announcement of icon-only action controls;
9. focus order after opening and dismissing menus;
10. whether actions remain reachable without touch.

Record the Android build, device/OEM, display topology, input device and accessibility service used. A pass on WSA or one physical device is useful evidence, but it is not a general Android compatibility claim.

## Reporting a problem

When reporting keyboard, pointer or accessibility problems, include:

- the exact surface (`Start`, taskbar, Settings, widget dialog, etc.);
- input method (mouse, touch, hardware keyboard, accessibility service);
- the key/button sequence;
- expected and observed focus/menu behavior;
- device and Android version;
- whether an external/secondary display is involved.

For broader device-support terminology, see [Compatibility and verification](COMPATIBILITY.md) and [Known limitations](KNOWN_LIMITATIONS.md). For shell/session behavior, see [Desktop sessions and shell modes](DESKTOP_SESSIONS.md).
