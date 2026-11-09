<!-- Reviewer record S01-R03, preserved as received. Only rendering artifacts were
     removed (favicon image links, HTML entities, a link-tracking query parameter);
     source attributions are kept as bracketed notes. Wording is otherwise unchanged. -->

# Reviewer Ruling — S01-R02: Architecture & Scope

ACCEPT

Project: Secure Mail Server Lab\
Review: S01-R02 → Reviewer disposition R02\
Date: November 9, 2026\
Reviewed commit: `91c56a6`\
Next authorized section: Section 2 — VMware Environment & Base OS

## 1. Final disposition

Section 1 is formally ACCEPTED. All four findings from R01 are closed at the architecture-design level.

I reviewed the committed [Builder correction](https://github.com/valeratech/secure-mail-platform/blob/91c56a6/reviews/S01-R02-builder-correction.md), along with `SCOPE.md` and `ENVIRONMENT.md`.

The revised design preserves the agreed architecture, incorporates the previously required security boundaries, and assigns implementation evidence to the appropriate later sections.

*[Reviewer source: GitHub (committed repository)]*

No further Section 1 revision is required.

## 2. Findings closure

| Finding                    | Decision     | Rationale                                                                               |
| -------------------------- | ------------ | --------------------------------------------------------------------------------------- |
| B1 — Network isolation     | Closed | Clear separation between isolated operation and declared maintenance                    |
| B2 — DNS and SMTP trust    | Closed | Explicit mail identity, sender simulation, anti-relay and destination restrictions      |
| B3 — Packages and versions | Closed | Verified package sources, version gates and appropriate Valkey correction               |
| B4 — CA, TLS and filtering | Closed | Defined certificate custody, listener policies, internal channels and offline filtering |

### Ruling A — Valkey replaces Redis: ACCEPT

The Builder's correction to R01 is technically correct.

Red Hat confirms that RHEL 10 replaced the Redis package with Valkey. More importantly, the official rspamd installation documentation now explicitly addresses EL10 and confirms that rspamd works with Valkey using the existing Redis configuration interface.

*[Reviewer source: Red Hat documentation; rspamd installation documentation]*

The proposed design is approved:

- Valkey installed through AlmaLinux AppStream.
- Local-only listener or Unix socket.
- Authentication enabled.
- No direct network access from the client VM.
- Integration tested with rspamd's stateful features.

Implementation verification remains assigned to Section 9.

### Ruling B — Two-tier private CA on client VM: ACCEPT

The proposed trust hierarchy is appropriate:

Root CA

Encrypted private key · offline custody

Intermediate CA

Certificate issuance · client VM administration

mail.lab.test

Leaf certificate + private key + intermediate chain

Operating the CA on the client VM is acceptable despite its role as an untrusted SMTP sender. SMTP trust and administrative trust are separate boundaries.

Section 4 implementation requirement: The root private key must genuinely be taken offline, rather than merely left encrypted on the running VM's filesystem. The intermediate signing key must also be access-restricted and excluded from the repository. Document how key custody interacts with VMware snapshots.

These are implementation safeguards, not new Section 1 blockers.

### Ruling C — Permanently attached, disconnected NAT adapter: ACCEPT

The proposed arrangement is approved.

Keeping NIC 2 permanently defined but virtually disconnected during normal operation maintains stable network-interface enumeration while allowing controlled package maintenance.

I particularly approve the three independent isolation mechanisms: disconnected virtual cable, no default route, and nftables outbound restrictions.

Section 2 implementation requirement: Verify that both `Connected` and `Connect at power on` are disabled for the NAT adapter during normal operation. Test this following a full shutdown and restart, not only a live disconnect.

During maintenance, permit required repository access without inadvertently exposing mail services or enabling outbound SMTP delivery.

## 3. Nonblocking implementation notes

These items are recorded for their respective sections. They do not reopen Section 1.

| Section | Reviewer carry-forward requirement                                                               |
| ------- | ------------------------------------------------------------------------------------------------ |
| 2       | Capture boot-time NIC state, network routes, repository trust and isolation evidence             |
| 3       | Verify authoritative DNS responses and rejected external recursion                               |
| 4       | Prove root-key offline custody, certificate-chain validity and client trust                      |
| 6       | Confirm submission restrictions cannot be bypassed during NAT maintenance                        |
| 9       | Prove rspamd-to-Valkey authentication and actual feature functionality, not just service startup |

One small correction for Section 6 documentation: submission port 587 must require TLS before authentication and mail submission, but the phrase “before any command but EHLO” is overly strict. Limited commands such as EHLO and STARTTLS, and certain other pre-TLS SMTP commands, are permitted by the protocol. The critical tests are refusal of pre-TLS authentication and mail submission.

I also accept the Builder's SPF/DKIM/DMARC negative-test matrix. In particular, scenario 2 correctly recognizes that DMARC can pass through aligned DKIM even when SPF fails.

## 4. Section 2 authorization

Section 1 — Architecture & Scope

Closed

Section 2 — VMware Environment & Base OS

Authorized

Builder is authorized to proceed with:

1. VMware VM creation and stable network interface design.
2. AlmaLinux 10.2 installation and baseline configuration.
3. Static addressing and host-only networking.
4. Baseline nftables and SELinux enforcement.
5. Package repository and maintenance-window procedures.
6. Evidence-backed isolation validation.

Governance: Section 1's accepted architecture is the baseline for subsequent sections. Later implementation failures are addressed within the section responsible for the failing component; they do not automatically reopen Section 1.

I verified that commit `91c56a6` and the referenced documents are publicly accessible. The reported green CI status has not been independently audited as part of this architectural review.

Final Reviewer decision: ACCEPT S01-R02. Section 1 closed. Section 2 unlocked.
