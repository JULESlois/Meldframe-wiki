# Architecture Decision Records

**Status:** normative process

Use ADRs for decisions that are expensive to reverse, define a public/stable boundary, or affect
multiple subsystems.

## Numbering

Use monotonically increasing files:

```
0001-example-title.md
0002-another-decision.md
```

## Lifecycle

`proposed → accepted → superseded / rejected`

An accepted ADR is evidence for *why* a design exists. If it changes an implementation contract,
update the corresponding main-repository documentation and tests as part of the same work.

## ADRs are not

- feature backlogs;
- meeting notes;
- research dumps;
- substitutes for API documentation.

Start from [0000-template.md](0000-template.md).
