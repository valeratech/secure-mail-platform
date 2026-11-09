# Scope — release one

Locked at Section 1 (accepted in [S01-R03](reviews/S01-R03-reviewer.md)). Anything not listed under "In scope" is deferred, including good ideas.
Changes to this file go through the Builder/Reviewer process in [reviews/](reviews/).

> **Status:** Section 1 **accepted** (S01-R03, 2026-11-09). Section 2 authorized. Later
> implementation failures are resolved within the section that owns the failing component;
> they do not reopen Section 1.

## Objective

A secure, production-style mail platform — SMTP, IMAP, webmail, filtering, and email
authentication — built by hand on AlmaLinux in an isolated VMware Workstation lab. The goal
is production-grade *administration practice*, not a publicly reachable production server.

## Locked decisions

| Area | Decision |
|---|---|
| OS | AlmaLinux 10.2 (RHEL-family, production parity) |
| Topology | Two VMs: `mail.lab.test` (full stack + BIND) and `client.lab.test` |
| Network | Custom host-only vmnet `10.10.10.0/24`; network-isolated in normal operation — **not** air-gapped |
| Namespace | `.test` (RFC 6761); email domain `lab.test`, synthetic sender domain `extsender.test` |
| MTA | Postfix |
| IMAP, auth, delivery | Dovecot 2.3 (2.3 syntax only); SASL authority for Postfix; LMTP into Maildir |
| User model | Virtual users in MariaDB |
| Filtering | rspamd as a milter, with Valkey (Redis-protocol) for stateful features |
| Webmail | Roundcube (EPEL) on Apache httpd + PHP |
| DNS | BIND, authoritative only, recursion disabled |
| TLS | Two-tier private CA; root key offline; mail VM holds leaf + chain only |
| Firewall | Native nftables as sole ruleset authority; firewalld disabled |
| MAC | SELinux enforcing throughout; never disabled to make a service work |
| Packages | Signed repositories only (AlmaLinux, EPEL, rspamd upstream); connected provisioning → verified isolation |

Known, accepted trade-off: BIND co-located on the mail VM is a single point of failure. It
is not represented as a fault-isolated production topology.

## In scope — 13 sections

| # | Section | Acceptance criteria (Reviewer additions in R01 §5 included) |
|---|---|---|
| 1 | Architecture & scope | This file, ENVIRONMENT.md, Reviewer acceptance |
| 2 | VMware environment & base OS | Isolation proof; **baseline nftables**; repository lifecycle (signed repos, maintenance windows, re-isolation test) |
| 3 | Internal DNS | Forward and reverse zones; `extsender.test` synthetic authentication domain; recursion disabled and verified |
| 4 | Private CA & TLS | Offline root-key custody; leaf with SAN; chain and trust verified from the client |
| 5 | MariaDB & virtual-user schema | Privilege separation per consuming service; strong password hashing; backup model |
| 6 | Postfix | Explicit relay policy; loopback-only `mynetworks`; sender authorization; restricted destinations; per-port TLS |
| 7 | Dovecot | Version-correct (2.3) configuration; permissions; SELinux; SASL and LMTP sockets |
| 8 | Core mail-flow checkpoint | Positive **and** negative SMTP/IMAP tests |
| 9 | rspamd | Offline limitations documented and enforced; Valkey; milter integration; DKIM signing and validation; B2 test matrix |
| 10 | Roundcube | Secure application configuration and authentication tests, independent of mail-protocol tests; DB privileges |
| 11 | Hardening | Full nftables policy; fail2ban (nftables action) verified; postscreen tuning |
| 12 | Validation | Evidence-based acceptance tests, including all negative tests |
| 13 | Operations | Backup **and restore drills** (mail, SQL, zones, certificates, config); updates; monitoring; incident procedures |

### Negative tests required by release one

Rejected unauthorized relay · failed authentication · AUTH refused before TLS on 587 ·
invalid TLS trust · nonexistent recipients · sender-login mismatch · out-of-domain
submission · SPF/DKIM/DMARC failure cases (B2 matrix) · access attempts to restricted
services (rspamd UI, MariaDB, Valkey, IMAP 143) from the lab network.

## Carry-forward requirements

Recorded by the Reviewer at Section 1 acceptance (S01-R03). Each is an acceptance criterion
of the section named, in addition to the table above. None reopens Section 1.

| Section | Requirement |
|---|---|
| 2 | NAT adapter has **both** "Connected" and "Connect at power on" disabled in normal operation, proven after a full shutdown and cold boot — not only a live disconnect |
| 2 | Evidence captured: boot-time NIC state, routing table, repository trust (key fingerprints), isolation proof |
| 2 | Maintenance windows permit repository access only — no exposure of mail services, no outbound SMTP |
| 3 | Authoritative answers verified; external recursion verified refused |
| 4 | Root key **genuinely offline** — not merely encrypted on the running VM's disk |
| 4 | Intermediate signing key access-restricted and excluded from the repository |
| 4 | Key custody interaction with VMware snapshots documented (a snapshot taken while a key is present preserves that key) |
| 4 | Certificate-chain validity and client trust proven |
| 6 | Submission restrictions proven not bypassable during a NAT maintenance window |
| 6 | **Correction to S01-R02:** port 587 requires TLS before AUTH and before mail submission. "Before any command but EHLO" overstated it — EHLO, STARTTLS and certain other commands are legitimately allowed pre-TLS. The tests are: AUTH refused before TLS, and MAIL/RCPT refused before TLS |
| 9 | rspamd → Valkey authentication proven, and stateful features (Bayes, greylisting, rate limiting) proven working — not only service startup |

## Explicitly out of scope

- Sending to or receiving from the public Internet; registered domains; public DNS
- Let's Encrypt or any public CA
- DNS blocklists, external reputation, public fuzzy storage, real-world deliverability
- High availability, clustering, a separate DNS VM, any third VM
- Groupware (calendar, contacts), mailing-list management, POP3
- Antivirus scanning
- `ARCHITECTURE.md`, diagrams, threat/control matrix — **derived after the build**, per
  portfolio conventions

## Verification standard

`postfix check`, `doveconf -n`, `named-checkconf`, and `rspamadm configtest` prove the
configuration parsed. They do not prove a control works. Each control in release one is
verified by demonstrating the attack or misuse **failing**: the relay rejected, the
cleartext AUTH refused, the spoofed sender quarantined or rejected, the off-lab packet
dropped.
