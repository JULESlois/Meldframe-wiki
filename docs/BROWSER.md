# Browser

**Audience:** users and testers  
**Status:** user documentation for the current development build

Meldframe includes an internal WebView-based browser. It is currently a practical shell application,
not the proposed future Browser Broker or an attempt to replace a full desktop browser engine.

## Current verified behavior

The current implementation has been exercised on the project's WSA test environment with:

- multiple tabs, with one WebView kept alive per tab;
- a tab strip integrated into the window title bar;
- new-tab, tab switching and tab closing;
- back, forward and refresh navigation;
- an address/search field;
- Meldframe's own local new-tab page;
- normal Meldframe minimize, maximize/restore and close controls.

The recorded WSA acceptance run verifies that a new tab opens next to the current tab and becomes
active, switching tabs updates the visible page/address state, closing the active tab selects a
neighbour, and closing the final tab closes the Browser window.

This is evidence for that tested environment, not a broad Android WebView/OEM compatibility claim.

## Tabs

Each tab owns a WebView and remains alive while the Browser window exists. Returning to another tab
therefore does not intentionally reconstruct that tab from scratch.

Tab placement and close behavior are deterministic:

- a tab opened from another tab is inserted next to its opener;
- closing the active tab prefers the tab to its right;
- if there is no right-hand neighbour, the tab to the left is selected;
- closing the final tab closes the Browser window.

The tab strip is also the Browser title bar. The unused area between tabs and the window controls is
the window drag surface.

## New-tab page

A blank Browser launch opens a Meldframe-owned local new-tab page rather than bundling another
browser vendor's start page. The page is stored with the application and intentionally uses ordinary
links and a GET search form rather than requiring page JavaScript.

This should not be confused with offline browsing: following links and searching still require the
corresponding network access.

## Address and search behavior

The address field distinguishes navigable URI-like input from search text. Current implementation
specifically handles cases such as ordinary HTTPS URLs, `about:blank`, localhost with a port, and
other URI schemes instead of recognizing only a short hard-coded scheme list.

`javascript:` input is refused from the address bar. This is an intentional safety boundary rather
than an unsupported-URL bug.

If an address is interpreted differently from what you expected, capture the exact text entered and
the resulting address before assuming that WebView itself failed.

## What this Browser does not prove

Current Browser work does **not** establish that Meldframe already provides:

- a Chromium extension platform;
- desktop-Chrome feature parity;
- browser profile/account synchronization;
- a stable automation API;
- Playwright or Puppeteer integration;
- the proposed Browser Broker;
- a general Web runtime for third-party Meldframe plugins.

Those are separate architecture or roadmap topics. In particular, the proposed Browser Broker would
provide a controlled browser service boundary; the existence of the current Browser UI does not mean
that broker exists today.

## Troubleshooting

If a page does not open:

1. Verify that ordinary HTTPS navigation works first.
2. Check whether the text was treated as an address or as a search query.
3. For a local service such as code-server, troubleshoot its Runtime/Service path separately rather
   than treating the Browser as proof that the backend service is healthy.
4. If only a particular site fails, remember that Android System WebView version, site policy,
   certificates, network configuration and device environment can all matter.

For Code, use the dedicated [Code guide](CODE.md); its WebView is a service-backed application path,
not simply a Browser tab.

## Related documentation

- [Using Meldframe](USAGE.md)
- [Compatibility and verification](COMPATIBILITY.md)
- [Known limitations](KNOWN_LIMITATIONS.md)
- [Code / service-backed apps](CODE.md)
- [Architecture overview](ARCHITECTURE.md)
