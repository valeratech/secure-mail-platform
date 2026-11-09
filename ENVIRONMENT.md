# Lab environment

Private VMware Workstation Pro lab on a Windows host. Two VMs on a custom host-only network.
Network-isolated during normal operation; **not** physically air-gapped.

> Installed package versions and repository key fingerprints are recorded here as each
> section installs them. Until then, the version tables are intentionally empty.

## Topology

```text
                 VMware custom host-only vmnet — 10.10.10.0/24, DHCP off

   Windows host                 ┌──────────────────────────────────────────────┐
   10.10.10.1 ─ management ────▶│ mail.lab.test             10.10.10.10        │
   (never a gateway)            │                                              │
   browser / IMAP client        │ Postfix   25 · 465 · 587                     │
                                │ Dovecot   993  (+ SASL, LMTP unix sockets)   │
                                │ httpd + Roundcube   443                      │
                                │ BIND      53   (authoritative, no recursion) │
                                │ rspamd · Valkey · MariaDB   loopback/sockets │
   ┌────────────────────────┐   │                                              │
   │ client.lab.test        │──▶│ nftables: ingress + egress allowlist         │
   │ 10.10.10.20  (+ .21)   │   │ SELinux enforcing                            │
   │ swaks · dkimsign       │   └──────────────────────────────────────────────┘
   │ openssl · private CA   │
   │ untrusted SMTP sender  │
   └────────────────────────┘
```

## VM sizing

Sized to keep host load modest. The client runs without a desktop; webmail and IMAP-client
testing happen from the host over the management path.

| | Mail VM | Client VM |
|---|---|---|
| vCPU | 2 | 1 |
| RAM | 4 GB | 1.5 GB |
| Disk | 40 GB thin | 20 GB thin |
| Install profile | Minimal | Minimal |

## Network adapters

Both VMs carry **two adapters permanently**, so device enumeration and interface names
never change between states:

| NIC | VMware network | Normal operation | Maintenance window |
|---|---|---|---|
| 1 | Lab host-only vmnet | Connected, static address, **no gateway** | Connected |
| 2 | NAT | **Present; "Connected" and "Connect at power on" both unchecked** | Connected (DHCP, default route) |

A maintenance window (provisioning, updates) is declared in the build log, opened by
connecting NIC 2 and loading the maintenance nftables set, and closed by disconnecting
NIC 2, restoring the baseline set, and **re-running the isolation proof**.

The disconnected state is proven after a **full shutdown and cold boot**, not only a live
disconnect — "Connect at power on" silently reconnects the cable at the next boot.

The maintenance nftables set opens outbound access to package repositories only (HTTP/HTTPS,
plus name resolution for them). Outbound SMTP stays blocked, and no mail service is exposed
on the NAT interface, during maintenance as at all other times.

### Isolation layers

1. Virtual cable — NIC 2 disconnected.
2. Routing — no default route on the lab connection.
3. Egress filter — mail VM `output` policy permits loopback and `10.10.10.0/24` only.
4. DNS — authoritative lab zones only; off-lab names refused.

### Host requirements

- No Internet Connection Sharing on the lab vmnet adapter.
- IP forwarding disabled on the lab vmnet adapter.
- VMware DHCP disabled for the lab vmnet.

## Addressing and names

| Address | Name | Role |
|---|---|---|
| `10.10.10.1` | — | Host management address (not a gateway) |
| `10.10.10.10` | `mail.lab.test` | Mail platform, authoritative DNS |
| `10.10.10.20` | `client.lab.test` | Untrusted sender, test tooling, private CA |
| `10.10.10.21` | — (no PTR) | Client alias: SPF-unauthorized source for negative tests |

Static addressing via NetworkManager keyfiles. Guests resolve against `10.10.10.10`.

## DNS zones

| Zone | Purpose |
|---|---|
| `lab.test` | The platform's mail domain: A, MX, SPF, DKIM, DMARC |
| `extsender.test` | Synthetic independent sender: SPF (`10.10.10.20` only), DKIM, DMARC |
| `10.10.10.in-addr.arpa` | PTR for `.10` and `.20` |

## Exposed services

On `10.10.10.10`, everything else is dropped:

| Port | Service | Reachable from |
|---|---|---|
| 22/tcp | SSH (keys only) | Lab subnet |
| 25/tcp | SMTP | Lab subnet |
| 465/tcp | Submission, implicit TLS | Lab subnet |
| 587/tcp | Submission, STARTTLS required | Lab subnet |
| 993/tcp | IMAPS | Lab subnet |
| 443/tcp | Roundcube | Lab subnet |
| 53/tcp+udp | DNS | Lab subnet |

## Private CA

```text
lab root CA  ──signs──▶  intermediate CA  ──signs──▶  mail.lab.test leaf
```

Operated on the client VM under a dedicated account. Root key passphrase-encrypted and
offline after signing the intermediate. The mail VM holds only the leaf, its key, and the
chain (leaf + intermediate). The root is trusted by the client VM and the host browser.
No CA state is ever committed.

## Package sources

| Repository | Used for | Key fingerprint |
|---|---|---|
| AlmaLinux 10 BaseOS / AppStream | OS, Postfix, Dovecot, BIND, MariaDB, httpd, PHP, Valkey | *recorded in Section 2* |
| EPEL 10 | Roundcube, fail2ban, swaks, python3-dkimpy | *recorded in Section 2* |
| rspamd upstream (EL10) | rspamd | *recorded in Section 9* |

`gpgcheck=1` everywhere. No `--nogpgcheck`.

## Installed versions

| Component | Version | Recorded in |
|---|---|---|
| AlmaLinux | | Section 2 |
| BIND | | Section 3 |
| MariaDB | | Section 5 |
| Postfix | | Section 6 |
| Dovecot (must be 2.3.x) | | Section 7 |
| rspamd | | Section 9 |
| Valkey | | Section 9 |
| httpd / PHP | | Section 10 |
| Roundcube | | Section 10 |
| fail2ban | | Section 11 |

## Snapshots

Stage boundaries only, matching the sections in [SCOPE.md](SCOPE.md). Architectural
snapshots are taken powered off, on both VMs, with the same name:

```text
02-base-os-isolated-clean
03-dns-clean
04-tls-clean
05-mariadb-schema-clean
06-postfix-clean
07-dovecot-clean
08-mail-flow-verified-clean
09-rspamd-clean
10-roundcube-clean
11-hardened-clean
12-validated-clean
```

Snapshots are section rollback points. They are **not** backups; the tested
backup-and-restore process is Section 13.
