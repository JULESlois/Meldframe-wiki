# Meldframe Wiki

This repository is Meldframe's engineering knowledge base: research, ideas, architecture reasoning,
development rules and decision records.

It is deliberately separate from the implementation repository.

## What belongs here

- **Research** — external systems, papers, source-code studies, experiments and comparisons.
- **Ideas** — hypotheses and possible features that are not commitments.
- **Development** — engineering principles and working rules for Meldframe contributors.
- **Decisions** — ADR-style records explaining why an architectural choice was accepted or rejected.
- **Sources** — curated primary references worth revisiting.

## What does not belong here

The implementation repository remains authoritative for code, executable tests, release state and
contracts that current code must obey. In particular, `docs/ROADMAP.md`, architecture contracts and
API documentation in the main repository describe the current implementation plan.

The wiki should not silently become a second source of truth.

## Knowledge flow

```
source / experiment
       ↓
research note
       ↓
idea / proposal
       ↓
ADR / accepted decision
       ↓
main-repository contract + tests
```

A useful idea is not automatically a roadmap commitment. An accepted decision that affects code must
eventually be reflected in the implementation repository.

## Document status

Every substantial note should state one of:

- `research` — evidence and analysis, no commitment.
- `idea` — deliberately speculative.
- `proposal` — concrete design under consideration.
- `accepted` — architectural decision has been accepted.
- `normative` — contributor rule that should be followed.
- `obsolete` — kept for history, not current guidance.

Prefer primary sources, source code and reproducible experiments over summaries.

## Initial index

- [ChromeOS, Android and Meldframe](research/chromeos-android-meldframe.md)
- [Ideas backlog](ideas/README.md)
- [Development principles](development/PRINCIPLES.md)
- [Architecture Decision Records](decisions/README.md)
- [Source index](sources/README.md)

## Project

Main implementation: https://github.com/JULESlois/Hyperdroid-recovery

The historical repository name remains for now; the forward product/project name is **Meldframe**.
