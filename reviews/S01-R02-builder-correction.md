# S01-R02 — Builder correction: Section 1 (Architecture & Scope)

| | |
|---|---|
| Responds to | [S01-R01 — Reviewer assessment](S01-R01-reviewer.md), disposition **REVISE** |
| From → To | Builder (Claude) → Reviewer |
| Date | 2026-11-09 |
| Requests | Section 1 acceptance; Section 2 remains gated on it |
| Authoritative result | [SCOPE.md](../SCOPE.md) and [ENVIRONMENT.md](../ENVIRONMENT.md) carry the locked state below |

All decisions accepted in R01 are preserved unchanged: AlmaLinux 10.2, two VMs, Postfix +
Dovecot + LMTP, MariaDB virtual users, rspamd, Roundcube + Apache, BIND on the mail VM,
`.lab.test`, private CA, SELinux enforcing, raw nftables, and the 13-section roadmap. No VM
is added, no component is replaced, and public Internet mail stays out of scope.

This revision resolves B1–B4, records three Builder corrections to my own Section 1 text,
and records one correction to R01 (B3, Redis → Valkey) for the Reviewer's ruling.

---

## Builder corrections to my own Section 1

| # | Section 1 said | Corrected to | Reason |
|---|---|---|---|
| C1 | The lab is "air-gapped" | **Network-isolated during normal operation, not air-gapped** | The host-only adapter at `10.10.10.1` is a live path to the host. R01 is right; the word overstated the boundary. |
| C2 | Roundcube "likely pulled current from upstream" | **EPEL package** | Accepting R01's recommendation — see B3. |
| C3 | rspamd features implied to work as on an Internet-connected host | **Explicit online/offline capability split** | See B4. Several rspamd modules depend on Internet services and would silently time out. |

---

## B1 — Isolation boundary

**Resolution: accepted in full, with one mechanism added to reconcile portfolio precedent.**

### Network definition

| Item | Value |
|---|---|
| Lab network | Dedicated VMware **custom host-only** vmnet, `10.10.10.0/24` |
| VMware DHCP on that vmnet | **Disabled** |
| `10.10.10.1` | Host virtual adapter — **management address only, never a gateway** |
| `mail.lab.test` | `10.10.10.10`, static |
| `client.lab.test` | `10.10.10.20`, static; `10.10.10.21` secondary alias (B2 negative tests) |
| Default route on guests (isolated mode) | **None** |
| NAT, bridging, ICS, host forwarding (isolated mode) | **None** |

### Reconciling with portfolio precedent

`secure-nginx-apache-platform` keeps a NAT adapter permanently attached so that boot-time
device state never changes between exercises. That reasoning is sound and worth keeping;
R01's requirement is also correct for a mail lab. Both are satisfied by:

- Each VM has **two adapters permanently present**: NIC 1 host-only (lab), NIC 2 NAT
  (maintenance).
- NIC 2 is **present but "Connected" unchecked** in VMware during normal operation. Device
  enumeration and interface names never change; only the virtual cable state does.
- NIC 2 is connected **only** inside a declared maintenance window (provisioning, updates),
  recorded in the build log, and disconnected again before the isolation test is re-run.

### Defense in depth

Isolation does not rest on the virtual cable alone:

1. **Virtual cable** — NIC 2 disconnected.
2. **Routing** — no default route on the lab connection (NetworkManager keyfile, no gateway).
3. **Egress filter** — nftables `output` policy on the mail VM permits only loopback and
   `10.10.10.0/24` in isolated mode. Maintenance mode is a separate, explicitly loaded rule
   set, not an edit to the baseline.
4. **DNS** — BIND answers only for lab zones with recursion disabled (B2), so off-lab names
   fail immediately rather than leaking queries or timing out.

### Host-side requirements (Windows host)

- The VMware host-only adapter for the lab vmnet has no Internet Connection Sharing.
- IP forwarding is disabled on that adapter.
- The host is never used as a router for the lab subnet.

### Isolation proof (Section 2 acceptance test)

Run from each VM, with NIC 2 disconnected; output captured as evidence:

| Check | Expected |
|---|---|
| Ping the other lab VM | Succeeds |
| Ping `10.10.10.1` from a VM | Succeeds (management path exists — documented, not hidden) |
| Routing table | No default route |
| Ping / TCP 443 to `8.8.8.8` | Fails — no route |
| Resolve a public name via the lab resolver | Fails — REFUSED, immediate |
| Host reaches guests on management ports | Succeeds |
| Host forwarding state on the lab adapter | Disabled |

The test is re-run after **every** maintenance window, not only once.

---

## B2 — DNS and SMTP security model

**Resolution: accepted in full.**

### Identity model

| Purpose | Value |
|---|---|
| Mail server FQDN / SMTP banner / HELO | `mail.lab.test` |
| Email domain | `lab.test` |
| Mailbox form | `alice@lab.test`, `bob@lab.test` |
| MX for `lab.test` | `mail.lab.test` |
| Reverse DNS | `10.10.10.10` → `mail.lab.test` (forward-confirmed) |
| DNS server | `10.10.10.10` (BIND, on the mail VM) |
| Synthetic independent sender | `extsender.test` — a separate domain with its own SPF/DKIM/DMARC |

### Zones (BIND, authoritative only)

| Zone | Records |
|---|---|
| `lab.test` | SOA, NS, A (`mail`, `client`), MX, SPF (TXT), DKIM (TXT, selector published in Section 9), `_dmarc` (TXT) |
| `extsender.test` | SOA, NS, SPF (TXT) authorizing `10.10.10.20` only, DKIM (TXT), `_dmarc` (TXT) |
| `10.10.10.in-addr.arpa` | PTR for `.10`, `.20` (`.21` deliberately **without** PTR, for negative tests) |

### Resolver policy

- **Recursion disabled.** BIND is authoritative for lab zones only; no forwarders and no
  root hints in use. Off-lab names return REFUSED immediately.
- Queries accepted from loopback and `10.10.10.0/24` only. Zone transfers denied.
- Listens on `127.0.0.1` and `10.10.10.10` only.
- Name resolution during maintenance windows (package downloads) is a Section 2 detail and
  must not leak lab-zone queries off-lab.

### Postfix trust boundary — corrected

- `mynetworks` = **loopback only**. The client VM is **not** trusted by address, even though
  it shares the subnet. It behaves as an untrusted remote MTA on port 25.
- Port 25: SASL authentication is **not offered**. Mail is accepted only for `lab.test`.
- Relay control uses explicit `smtpd_relay_restrictions`
  (`permit_mynetworks, permit_sasl_authenticated, reject_unauth_destination`), with
  `smtpd_recipient_restrictions` as a separate policy layer — not recipient restrictions
  alone.

### Bounding authenticated submission

Two controls, so an authenticated account cannot become an outbound path:

1. **Sender authorization** — `reject_sender_login_mismatch` against an SQL
   sender-login map. An authenticated user may only send as addresses they own.
2. **Restricted destinations** — at RCPT time on 587/465, recipients outside `lab.test`
   are rejected with a clear 5xx. As a second layer, Postfix's default and relay transports
   are set to an error transport, so even a policy mistake cannot hand mail to a remote
   delivery agent. Delivery is local-only by construction.

### Independent-sender simulation (no extra VM)

The client VM originates mail as `extsender.test` into `mail.lab.test:25`:

- `swaks` drives the SMTP sessions.
- `dkimsign` (python3-dkimpy) signs messages with the `extsender.test` key, which lives on
  the client VM only.
- `10.10.10.21` provides an address **not** authorized by `extsender.test` SPF.

| # | Scenario | Expected verdict |
|---|---|---|
| 1 | `extsender.test`, from `.20`, valid DKIM | SPF pass, DKIM pass, DMARC pass |
| 2 | `extsender.test`, from `.21`, valid DKIM | SPF fail, DKIM pass, **DMARC pass** (DKIM-aligned) |
| 3 | `extsender.test`, from `.21`, no DKIM | SPF fail, DMARC fail → policy applied |
| 4 | Body altered after signing | DKIM fail |
| 5 | Unknown DKIM selector | DKIM permerror / fail |
| 6 | Client spoofs `lab.test` on port 25 | SPF fail, DMARC fail → policy applied |
| 7 | Relay attempt to a non-lab domain on port 25 | Rejected at RCPT |
| 8 | Authenticated user sends as another user | Rejected (sender login mismatch) |

Scenario 2 is included on purpose: it is the case most often misread as a DMARC failure.

---

## B3 — Package and version assumptions

**Resolution: accepted, with one correction to R01 (Redis → Valkey).**

Versions are **recorded from `dnf info` at install time** and written into ENVIRONMENT.md.
Nothing below is a version promise; it is the source decision and the verification gate.

| Component | Source | Builder finding | Gate |
|---|---|---|---|
| Postfix | AlmaLinux 10 BaseOS/AppStream | Distribution package | Record at install (S6) |
| Dovecot | AppStream | **2.3.21 confirmed** on EL10 rebuilds (`2.3.21-16.el10`). Configuration follows **2.3 syntax only**; 2.4 documentation is out of bounds. | Record at install; reject 2.4 directives (S7) |
| BIND | AppStream | Distribution package | Record at install (S3) |
| MariaDB | AppStream default stream | Version and connector extensions pinned | Record at install (S5) |
| httpd + PHP | AppStream | PHP version and the extensions Roundcube requires pinned | Compatibility check (S10) |
| Roundcube | **EPEL 10** | Accepting R01. Builder could not independently confirm the exact EPEL build R01 cites; `dnf info` at install is the authority. EPEL tracks a steady stream of 2026 Roundcube CVEs, so a dnf-managed patch path outweighs upstream freshness. | Version + advisory check (S10) |
| rspamd | Upstream rspamd RPM repository (EL10) | Not in base or EPEL | GPG key fingerprint recorded (S9) |
| **Valkey** | AppStream | See correction below | Record at install (S9) |
| fail2ban | EPEL | Native nftables ban action (R01 ruling 9b) | S11 |
| swaks, python3-dkimpy | EPEL (client VM) | Test tooling only | S8 / S9 |

### Correction to R01 — Redis is not available on EL10

R01 recommends Redis for rspamd's Bayes, greylisting, rate limiting, and DMARC reporting.
The requirement is right; the package is not. **RHEL 10 replaced Redis with Valkey**, and
AlmaLinux 10 follows: Redis is absent from the supported repositories, while Valkey ships
in AppStream. Valkey is a protocol-compatible fork, so rspamd's Redis backend uses it
unchanged.

**Decision:** install Valkey on the mail VM, bound to loopback (or a unix socket) only,
with authentication. Bayes, greylisting, rate limiting, and DMARC aggregate reporting stay
**in scope**. Building Redis from source is rejected — it would bypass the signed-package
update lifecycle this revision exists to establish.

### Repository trust

`gpgcheck=1` on every repository. Each third-party key (EPEL, rspamd) is imported
deliberately and its fingerprint recorded in ENVIRONMENT.md before first use. No
`--nogpgcheck`, ever.

---

## B4 — Trust, TLS, and filtering boundaries

**Resolution: accepted in full.**

### Private CA custody

Two-tier CA, consistent with portfolio precedent:

```text
lab root CA  ──signs──▶  intermediate CA  ──signs──▶  mail.lab.test leaf
```

- The CA is operated on the **client VM**, under a dedicated account. The client VM is
  "untrusted" only in the SMTP-relay sense; it is also the lab's admin workstation.
- The **root key** is passphrase-encrypted and taken offline after it signs the
  intermediate. It is used again only to re-issue the intermediate.
- The **mail VM receives only** the leaf certificate, the leaf key, and the chain file
  (leaf + intermediate). It never holds the root or intermediate key.
- The leaf carries `subjectAltName = DNS:mail.lab.test`. The root is installed in the
  client VM's trust store and in the host browser used for webmail testing.
- No CA state or private key enters git — enforced by `.gitignore`, the audit, and gitleaks.

*Alternative for the Reviewer:* CA on the Windows host instead of the client VM. Rejected
by the Builder because it puts key material on the same machine that stages commits.

### TLS behavior per listener

| Port | Service | TLS | Authentication |
|---|---|---|---|
| 25 | SMTP (MTA-to-MTA) | **Opportunistic** STARTTLS offered; cleartext accepted | Not offered |
| 587 | Submission | STARTTLS **mandatory** before any command but EHLO | Required; only after TLS |
| 465 | Submission | **Implicit** TLS | Required |
| 993 | IMAPS | Implicit TLS | Required |
| 143 / 110 | IMAP / POP3 | **Not exposed** | — |
| 443 | Webmail | HTTPS | Roundcube login |
| 80 | HTTP | **Closed** | — |

Port 25 is deliberately different: a receiving MX that refuses cleartext rejects legitimate
senders. That trade-off is documented rather than "fixed." The protocol floor is TLS 1.2,
taken from the EL10 system-wide crypto policy; per-service overrides need a written reason.

### Internal service channels

| From → To | Channel |
|---|---|
| Postfix → Dovecot auth (SASL) | Unix socket inside the Postfix spool, Postfix-only permissions |
| Postfix → Dovecot delivery (LMTP) | Unix socket inside the Postfix spool |
| Postfix ↔ rspamd (milter) | `127.0.0.1` only |
| rspamd ↔ Valkey | Loopback or unix socket, authenticated |
| rspamd web UI | `127.0.0.1` only — reached by SSH tunnel |
| Postfix / Dovecot / Roundcube → MariaDB | Unix socket; MariaDB networking disabled |
| Roundcube → Dovecot / Postfix | Loopback IMAP and submission |

Externally reachable on `10.10.10.10`: 22 (lab subnet), 25, 465, 587, 993, 443, and 53
(lab subnet). Nothing else. SELinux contexts and booleans for each channel are defined in
the section that creates it.

### rspamd in an offline lab

| Works offline (in scope, tested) | Does not work offline (excluded, disabled) |
|---|---|
| SPF, DKIM, DMARC, ARC verification against lab DNS | DNS blocklists (RBL / SURBL / URIBL) |
| DKIM signing for `lab.test` | Public fuzzy-hash storage |
| Bayes classification (Valkey) | External reputation and URL reputation |
| Greylisting, rate limiting (Valkey) | Automatic map / rule updates from upstream |
| DMARC aggregate report generation (delivered to a lab mailbox) | Sending DMARC reports to external domains |
| Local rules, multimap, MIME checks | Real-world deliverability |

Network-dependent modules are **disabled explicitly**, not left to fail: a module that
times out on every scan inflates latency and produces misleading "clean" verdicts.

---

## Section acceptance criteria

R01's per-section additions (R01 §5) are adopted without change into SCOPE.md. R01 §4
safeguards are placed as follows: baseline nftables in **Section 2**; SELinux enforcing
throughout; snapshots ≠ backups, with restore drills in **Section 13**; Roundcube security
tested independently of mail-protocol tests in **Section 10**; negative tests across
**Sections 8 and 12**.

## Requested disposition

**ACCEPT** Section 1, with a ruling on the Valkey correction to R01 (B3). On acceptance,
the Builder proceeds to Section 2 — VMware environment, base OS, isolation proof, baseline
nftables, and repository lifecycle.
