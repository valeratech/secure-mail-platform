<!-- Reviewer record S01-R01, preserved as received. Only rendering artifacts were
     removed (favicon image links, HTML entities); source attributions are kept as
     bracketed notes. Content and wording are otherwise unchanged. -->

# Reviewer Assessment — Section 1: Architecture & Scope

REVISE — Limited architectural corrections

Project: Secure Mail Server Lab\
Review date: November 9, 2026\
Builder: Claude\
Reviewer: ChatGPT\
Checkpoint: Section 1 → Section 2

## 1. Executive assessment

The proposed architecture is technically sound as a foundation for a secure, production-style mail infrastructure lab.

I support retaining the two-VM design, AlmaLinux 10.2, Postfix, Dovecot, rspamd, MariaDB, Roundcube, BIND, SELinux enforcing, and raw nftables.

The Builder has also made the correct distinction between practicing production-grade administration and operating a publicly reachable production mail server.

However, I am not granting unconditional Section 1 sign-off yet. Several technical boundaries need to be made explicit before VM provisioning begins.

The principal concerns are network isolation, email authentication testing, DNS design, and the software installation/update lifecycle.

### Architecture decision matrix

| Component                         | Reviewer decision           |
| --------------------------------- | --------------------------- |
| AlmaLinux 10.2                    | Accept                      |
| Two-VM topology                   | Accept                      |
| Postfix + Dovecot + LMTP          | Accept                      |
| MariaDB virtual users             | Accept                      |
| rspamd                            | Accept                      |
| Roundcube + Apache                | Accept                      |
| BIND on mail VM                   | Accept with conditions      |
| `.lab.test` namespace             | Accept                      |
| Private CA                        | Accept with conditions      |
| SELinux enforcing                 | Accept                      |
| Raw nftables                      | Accept                      |
| Isolated VMware VMnet             | Revise isolation definition |
| Package/update strategy           | Must resolve in Section 2   |
| Mail authentication test coverage | Revise                      |

AlmaLinux 10.2 is a valid released version, and its release notes confirm the relevant RHEL-family platform, including BIND and nftables.

*[Reviewer source: AlmaLinux]*

Reviewer direction: Keep the agreed core architecture. Correct its incomplete boundaries rather than expanding the number of VMs or introducing unnecessary services.

## 2. Required architectural corrections

### B1 — Define the actual VMware isolation boundary

Required correction

The Builder describes the VMnet as fully isolated and air-gapped, but the proposed `10.10.10.1` host adapter creates connectivity between the host computer and the guests.

VMware host-only networking does not automatically guarantee isolation from a host that has routing or Internet Connection Sharing configured.

*[Reviewer source: VMware documentation]*

Reviewer decision: Retain host-only networking, including `10.10.10.1`, for convenient host-side administration.

Require these design constraints:

- `10.10.10.1` is a host-side management address, not a default gateway.
- Disable VMware DHCP and use static guest addressing.
- No default route, NAT, bridging, Internet Connection Sharing, or host forwarding during isolated operation.
- Verify that both VMs can communicate but cannot reach outside networks.
- State clearly that the lab is network-isolated during normal operation, not physically air-gapped.

This preserves convenience without misrepresenting the security boundary.

### B2 — Formalize the DNS and SMTP security model

The Builder needs to define the distinction between the mail hostname and the email domain.

| Purpose          | Proposed value   |
| ---------------- | ---------------- |
| Mail server FQDN | `mail.lab.test`  |
| Email domain     | `lab.test`       |
| Mailbox example  | `alice@lab.test` |
| SMTP hostname    | `mail.lab.test`  |
| DNS server       | `10.10.10.10`    |
| Mail exchanger   | `mail.lab.test`  |

The DNS design should include forward and reverse zones, with A, MX, PTR, SPF, DKIM, and DMARC records as appropriate.

`.test` and its subdomains are reserved for testing under RFC 6761.

*[Reviewer source: RFC Editor]*

Important security correction: Do not automatically trust the entire `10.10.10.0/24` subnet through Postfix `mynetworks`.

The `client` VM must be capable of behaving as an untrusted SMTP sender on port 25, even though it shares the local subnet.

Additionally:

- Use explicit `smtpd_relay_restrictions`, not only recipient restrictions, for anti-relay protection.

  *[Reviewer source: Postfix documentation]*

- Ensure authenticated submission cannot become a path to sending messages outside the approved internal domains.

- Define the BIND recursion policy and prevent external DNS resolution during isolated operation.

- Define an internal mechanism for simulating independent sending domains.

That last point matters because meaningful SPF, DKIM, and DMARC validation requires positive and negative test conditions.

No additional VM is required. A second synthetic DNS domain and SMTP test tooling on the client VM can provide useful independent-sender scenarios.

### B3 — Correct package/version assumptions

The AlmaLinux package assessment needs updating.

| Software      | Reviewer finding                                                                 |
| ------------- | -------------------------------------------------------------------------------- |
| Dovecot       | AlmaLinux 10.2 AppStream has Dovecot 2.3.21 packages; do not use 2.4-only syntax |
| Roundcube     | EPEL 10.2 has Roundcube 1.6.20, including a November 2026 security update        |
| rspamd        | An official EL10-compatible RPM repository is available                          |
| MariaDB / PHP | Versions and extensions need to be pinned to verified compatible packages        |

Sources: AlmaLinux repository index, Fedora EPEL security advisory, and rspamd upstream repository.

*[Reviewer source: AlmaLinux repository index, Fedora EPEL advisory, rspamd upstream]*

Reviewer recommendation: Prefer supported distribution/EPEL packaging for Roundcube unless the Builder documents a concrete reason for installing directly from upstream.

The Dovecot version distinction is particularly important because configuration syntax differs between 2.3 and 2.4.

### B4 — Define trust, TLS, and filtering boundaries

The security design should explicitly establish that:

- The private CA root signing key is maintained outside the mail server VM; the VM receives its issued certificate and required chain, not the root private key.
- SMTP submission on 587 uses mandatory STARTTLS, and 465 uses implicit TLS.
- IMAP on 993 and webmail on 443 require TLS.
- SMTP port 25 has its own documented TLS behavior; it must not be treated identically to authenticated submission.
- Postfix, Dovecot, rspamd, and MariaDB communicate over appropriately restricted local sockets or loopback connections.

For rspamd, the Builder should also distinguish what actually works in an offline lab. Locally defined SPF/DKIM/DMARC policies can be tested, but public DNS blocklists, external reputation lookups, and real-world deliverability cannot be validated without external connectivity.

One additional dependency deserves attention: Redis. Rspamd recommends it for production deployments because features such as Bayes classification, rate limiting, greylisting, and DMARC aggregate reporting depend on it.

*[Reviewer source: rspamd documentation]*

I recommend including Redis on the existing mail VM if those features are part of the required filtering scope. Otherwise, explicitly document their exclusion.

## 3. Rulings on the Builder's flagged decisions

9(a) Two-VM consolidation — ACCEPT

Combining BIND with the mail stack introduces a single point of failure, but is reasonable for a resource-conscious lab. It must not be represented as a resilient, fault-isolated production topology.

9(b) Raw nftables — ACCEPT

This is valid on RHEL-family systems. Red Hat documents both nftables and firewalld, while recommending that only one firewall-management service run on a host.

*[Reviewer source: Red Hat documentation]*

Use native nftables as the ruleset authority, disable firewalld, and integrate fail2ban through a compatible nftables ban action.

9(c) Package acquisition and updates — DEFER TO SECTION 2 WITH REQUIREMENTS

I favor a connected provisioning → verified isolation model over immediately building an entire offline repository mirror.

The Builder must document temporary Internet access, repository trust/signature verification, removal of connectivity, an isolation verification test, and the process for obtaining subsequent security updates.

## 4. Additional implementation safeguards

These are not reasons to redesign Section 1, but they must appear in the relevant implementation sections.

Baseline firewall: nftables protection begins during Section 2, rather than waiting until Section 11. Section 11 should be the advanced hardening phase.

SELinux: Keep enforcing throughout the project. Verify actual package policies, file contexts, and AVC denials before introducing local policy modifications. Never disable enforcement merely to make a service work.

Snapshots versus backups: VMware snapshots are suitable for section rollback, but are not substitutes for a tested backup-and-restore process. Section 13 must cover mailbox data, SQL databases, DNS zones, certificates and configuration, including consistent recovery.

Roundcube: Test its security configuration and database privileges independently of the mail protocol tests. A working web login does not establish a secure webmail deployment.

Negative testing: The final validation must demonstrate rejected unauthorized relay attempts, failed authentication, invalid TLS trust, nonexistent recipients, mail-policy violations, and attempted access to restricted services.

## 5. Roadmap assessment

The 13-section roadmap is accepted as the project's baseline structure.

I recommend keeping the sequence unchanged, with these adjustments to the section acceptance criteria:

| Section                  | Required addition                                                    |
| ------------------------ | -------------------------------------------------------------------- |
| 2 — VMware & OS          | Isolation proof, base nftables, repository lifecycle                 |
| 3 — DNS                  | Forward/reverse zones, synthetic authentication test domain          |
| 4 — CA & TLS             | Offline root-key custody, certificate verification                   |
| 5 — MariaDB              | Database privilege separation, password hashing, backup model        |
| 6 — Postfix              | Explicit relay policy, sender authorization, restricted destinations |
| 7 — Dovecot              | Version-correct configuration, permissions, SELinux and LMTP         |
| 8 — Mail-flow checkpoint | Positive and negative SMTP/IMAP tests                                |
| 9 — rspamd               | Offline limitations, Redis decision, Milter and DKIM validation      |
| 10 — Roundcube           | Secure application configuration and authentication tests            |
| 11 — Hardening           | Full policy integration and fail2ban verification                    |
| 12 — Validation          | Evidence-based acceptance tests                                      |
| 13 — Operations          | Restore drills, updates, monitoring, incident procedures             |

The roadmap itself does not need another restructuring exercise.

## 6. Reviewer disposition and handoff to Builder

Section 1 — Architecture & Scope

REVISE

Review R01 · November 9, 2026

Accepted: Core technology stack, two-VM deployment, AlmaLinux 10.2, architecture objectives, 13-section roadmap, SELinux and nftables choices.

Remaining: Incorporate B1–B4 as documented architectural constraints.

Next handoff: Builder submits one consolidated Section 1 correction summary, with the updated network/mail-flow boundaries and software-dependency decisions. Implementation of Section 2 remains gated on Section 1 acceptance.

### Instructions to Builder

Preserve the existing architecture and all accepted decisions. Do not add VMs, replace the mail stack, or expand the project into public Internet mail.

Resolve the four identified architecture corrections in one consolidated revision. Detailed commands, configuration files, and provisioning procedures belong to their respective future sections and are not required for this checkpoint.

Reviewer conclusion: The design is very close to approval. It needs tighter definitions around isolation, trusted SMTP clients, offline security testing, and service dependencies—not a fundamental architectural change.
