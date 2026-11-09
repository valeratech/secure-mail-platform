# Reviews

The Builder/Reviewer record for each section checkpoint. A section does not close, and the
next does not begin, until the Reviewer records **ACCEPT**.

## Protocol

- The **Builder** proposes a section and submits a summary for review.
- The **Reviewer** returns a disposition: **ACCEPT**, **REVISE**, or **REJECT**, with
  numbered findings.
- The Builder answers every finding in one consolidated revision — accepting it, or
  disagreeing with evidence for the Reviewer to rule on.
- Each round is committed **as written**. Corrections are new files, never edits to an
  earlier round, so the record shows what was proposed, what was challenged, and why it
  changed.
- The locked result of an accepted section lives in `SCOPE.md` / `ENVIRONMENT.md`; the files
  here are the reasoning that produced it.

## Naming

```text
S<section>-R<round>-<role>.md        e.g. S01-R01-reviewer.md, S01-R02-builder-correction.md
```

## Record

| File | Section | Role | Disposition |
|---|---|---|---|
| [S01-R01-reviewer.md](S01-R01-reviewer.md) | 1 — Architecture & scope | Reviewer | REVISE (B1–B4) |
| [S01-R02-builder-correction.md](S01-R02-builder-correction.md) | 1 — Architecture & scope | Builder | Submitted, requests ACCEPT |

The Builder's original Section 1 proposal was delivered in conversation and is summarized by
R01's assessment; R01 is the first committed round.
