# secure-mail-platform

A hand-built, production-style mail platform — Postfix, Dovecot, rspamd, MariaDB virtual
users, Roundcube, BIND, and a private CA — on AlmaLinux 10 in an isolated VMware
Workstation lab, with the reasoning, trust boundaries, and failure analysis documented
alongside the configuration.

## Why this exists

Mail is the one service where DNS, TLS, authentication, authorization, and content
filtering all have to agree before a single message is delivered correctly. Control panels
and hosted providers hide most of that agreement. This project builds it by hand, section
by section, so every boundary is visible: who is trusted to relay, who authenticates whom,
which component makes each accept/reject decision, and what each control actually stops.

The target is production-grade **administration practice** — not a publicly reachable
production mail server.

## Safety boundaries

- Isolated VMware host-only network (`10.10.10.0/24`), DHCP off, no gateway. Network-isolated
  during normal operation; Internet access exists only inside declared maintenance windows
  for signed package installation. **Not** described as air-gapped, because it is not.
- Reserved test namespace only (`lab.test`, `extsender.test` — RFC 6761). No registered
  domain, no public DNS, no mail to or from the Internet.
- No employer or production configuration, customer data, real domain, or real address
  appears anywhere in this repository.
- Private keys, CA state, DKIM keys, and credentials are generated in the lab and never
  committed.

## Repository layout

```text
SCOPE.md                   locked release-one scope, sections, acceptance criteria
ENVIRONMENT.md             topology, VM sizing, addressing, zones, snapshots, versions
reviews/                   Builder/Reviewer record — every section checkpoint, as written
lab/                       break/fix exercises — the deliberate failures
reference/                 the clean authoritative configuration (synced off the VM)
scripts/                   validation, audit, and evidence tooling
```

`reference/` is populated only by copying configuration off the running VM, never by typing
it into the repo. `lab/` contains the mistakes. They are kept separate on purpose.

## How it is built

Each section is proposed by the **Builder**, assessed by an independent **Reviewer**, and
accepted before the next begins. Review rounds are committed as written, including the
corrections, in [reviews/](reviews/). The stage-boundary commit history is itself evidence
that the build was staged.

Commit gates are adapted from the Linux portfolio hub's
[`WORKFLOW.md`](https://github.com/valeratech/linux-infrastructure-portfolio/blob/main/WORKFLOW.md):
redact at the source → custom audit → pre-commit hooks → CI full-history gitleaks scan.
This repository confirms CI with two corrected forms of that file's §7 check:
`gh run watch --exit-status`, so a failed run fails the step, and a commit-count match
that fits the pinned gitleaks output (`N commits scanned`).

```bash
pre-commit install            # once
./scripts/audit-sanitize.sh   # before staging anything
pre-commit run --all-files
```

`.audit-denylist` (gitignored) holds employer, customer, and production strings that must
never be committed — create it before the first commit.

Architecture documentation and the threat/control matrix are derived at the end, from what
was actually built and verified.

## Status

| # | Section | State |
|---|---|---|
| 1 | Architecture & scope | **Accepted** — [S01-R03](reviews/S01-R03-reviewer.md) |
| 2 | VMware environment & base OS | In progress |
| 3 | Internal DNS | |
| 4 | Private CA & TLS | |
| 5 | MariaDB & virtual-user schema | |
| 6 | Postfix | |
| 7 | Dovecot | |
| 8 | Core mail-flow checkpoint | |
| 9 | rspamd | |
| 10 | Roundcube | |
| 11 | Hardening | |
| 12 | Validation | |
| 13 | Operations | |

Break/fix progress is tracked in [lab/README.md](lab/README.md).
