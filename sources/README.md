# Source Index

**Status:** research

Prefer primary documentation and source code. Record the exact subsystem or claim a source supports
rather than adding links without context.

## ChromeOS

- ChromiumOS containers and VMs overview — Crostini components, lifecycle and host/guest integration:  
  https://www.chromium.org/chromium-os/developer-library/guides/containers/containers-and-vms/
- Crostini developer guide — component responsibilities and development details:  
  https://www.chromium.org/chromium-os/developer-library/guides/containers/crostini-developer-guide/
- Chromium App Service source/docs — unified application publishers/consumers and instance model:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/components/services/app_service/
- Exo — ChromeOS Wayland server used by ARC/Crostini integration:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/components/exo/README.md
- Sommelier — guest Wayland/X11 proxy and buffer strategies:  
  https://chromium.googlesource.com/chromiumos/platform2/+/HEAD/vm_tools/sommelier/README.md
- Lacros design/history — browser/OS boundary and the cost of physical/release separation:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/docs/lacros.md
- ChromeOS System Web Apps source/docs:  
  https://chromium.googlesource.com/chromium/src/+/HEAD/chrome/browser/ash/system_web_apps/

## Android

- Android Virtualization Framework use cases — including the Linux development environment:  
  https://source.android.com/docs/core/virtualization/usecases
- VirtualizationService / VM lifecycle:  
  https://source.android.com/docs/core/virtualization/virtualization-service
- Android desktop windowing architecture:  
  https://source.android.com/docs/core/display/desktop-windowing
- Android desktop UX / multitasking guidance:  
  https://developer.android.com/design/ui/desktop/guides/system/multi-task
- crosvm source and documentation:  
  https://chromium.googlesource.com/crosvm/crosvm/

## Termux

- Termux execution environment:  
  https://github.com/termux/termux-packages/wiki/Termux-execution-environment
- Building packages / custom package-name implications:  
  https://github.com/termux/termux-packages/wiki/Building-packages
- RUN_COMMAND integration:  
  https://github.com/termux/termux-app/wiki/RUN_COMMAND-Intent

## Research hygiene

For each future research note:

1. capture primary sources first;
2. record the relevant revision/date when behavior is fast-moving;
3. distinguish current facts from Meldframe proposals;
4. attach experiments or code references when a claim can be tested;
5. mark obsolete conclusions instead of silently rewriting history.
