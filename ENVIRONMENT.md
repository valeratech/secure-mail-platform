# Lab environment

Private VMware Workstation Pro lab on a Windows host. Two VMs on a custom host-only network.
Network-isolated during normal operation; **not** physically air-gapped.

> Facts are recorded here as each stage builds them. **Section 2 so far:** the host-side lab
> network (2.1), the verified installation media (2.2), the mail VM (2.3), and the client VM
> (2.4) are recorded below. Rows for later stages and sections stay empty until they are
> built.

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

### Mail VM as built (Section 2.3)

| Item | Value |
|---|---|
| Virtual hardware / guest type | Workstation 25H2 / Red Hat Enterprise Linux 10 64-bit |
| Firmware | **UEFI**. VMware offered no Secure Boot option for this VM (deviation D12). |
| CPU / RAM | 1 socket × 2 cores / 4 GB |
| Disk | 40 GB, NVMe, thin, single file (`nvme0n1`); SCSI controller PVSCSI (idle) |
| Removed devices | USB controller, sound card, printer; 3D acceleration off |
| Host-guest channels | Shared folders, drag-and-drop, copy/paste and VNC disabled; VMware Tools host time sync off |
| Install media | AlmaLinux 10.3 minimal ISO (verified in 2.2). The VM was configured to disconnect installation media after setup. Guest-side evidence continued to identify an optical device; media detachment remains unverified and will be checked during Stage 2.5. |

### Client VM as built (Section 2.4)

| Item | Value |
|---|---|
| Virtual hardware / guest type | Workstation 25H2 / Red Hat Enterprise Linux 10 64-bit |
| Firmware | **UEFI**; no Secure Boot option offered, as on the mail VM (D12) |
| CPU / RAM | 1 vCPU / **1.5 GB** (1536 MB). Installed at 2 GB for the graphical installer, then reduced while powered off; the guest reports 1425 MiB usable. |
| Disk | 20 GB, NVMe, thin, single file (`nvme0n1`); SCSI controller PVSCSI (idle) |
| Removed devices | USB controller, sound card; 3D acceleration off |
| Host-guest channels | Shared folders, drag-and-drop and copy/paste disabled; VMware Tools host time sync off |
| Guest tools | `open-vm-tools` is **not installed** (Minimal install). VMware's "Shut Down Guest" therefore cannot shut the client down cleanly; it is powered off from inside the guest. |
| Install media | AlmaLinux 10.3 minimal ISO (verified in 2.2). "Connect at power on" unchecked while powered off, then **verified from inside the guest** after a cold boot: `blkid` finds nothing, udev reports no media, and a read fails with "No medium found". The kernel still lists `sr0` with a placeholder size; that is the empty drive, not the ISO. |

### Client VM disk layout

The client stores no mail, so it uses the installer's automatic layout with no reserved
space.

| Volume | Size | Type |
|---|---|---|
| `/boot/efi` | 600 MiB | `nvme0n1p1`, vfat |
| `/boot` | 2 GiB | `nvme0n1p2`, XFS |
| `/` | 15.41 GiB | LV `almalinux_client/root`, XFS |
| swap | 2 GiB | LV `almalinux_client/swap` (sized from the 2 GB install-time RAM) |
| VG free | 0 | — |

### Mail VM disk layout (D7)

| Volume | Size | Type |
|---|---|---|
| `/boot/efi` | 600 MiB | `nvme0n1p1`, vfat |
| `/boot` | 2 GiB | `nvme0n1p2`, XFS |
| `/` | 23.5 GiB | LV `almalinux_mail/root`, XFS |
| swap | 3.91 GiB | LV `almalinux_mail/swap` |
| **VG free** | **10 GiB** | Reserved in VG `almalinux_mail` for a dedicated `/var/vmail` volume (Section 7) |

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

**Mail VM as built:** NIC 1 → guest `ens160`, NIC 2 → guest `ens192` (VMXNET3). The VM was
created with *no* network adapter and both were then added explicitly, lab adapter first, so
a missed step yields no network rather than Internet access and the lab NIC enumerates
first. While a VM is powered off VMware exposes only "Connect at power on" ("Connected" is a
runtime control); both are checked together in the 2.8 cold-boot proof.

**Client VM as built:** the same pattern: created with no adapter, lab adapter added first,
NIC 1 → `ens160` on VMnet10 with "Connect at power on" checked, NIC 2 → `ens192` on NAT with
"Connect at power on" unchecked. After a cold boot the guest reports `ens192` unavailable
(no carrier).

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

### Host network as built (Section 2.1)

| Item | Recorded state |
|---|---|
| Lab network | Custom host-only **VMnet10**, subnet `10.10.10.0/24`, VMware DHCP **off** |
| Host adapter | VMware Network Adapter VMnet10, `10.10.10.1/24`, **no gateway** |
| IPv4 forwarding on that adapter | **Disabled** |
| Internet Connection Sharing | No evidence of active ICS forwarding through VMnet10 was observed. Interface forwarding was disabled, and no ICS-associated private-interface addressing was identified. The behavioral egress test remains scheduled for Stage 2.8. |
| Host to guest path | Directly connected on VMnet10; ping to the mail VM returns TTL 64 undecremented (no routed hop) |

The Windows ICS COM query was inconclusive in a non-elevated session and is not relied on.

## Addressing and names

| Address | Name | Role |
|---|---|---|
| `10.10.10.1` | — | Host management address (not a gateway) |
| `10.10.10.10` | `mail.lab.test` | Mail platform, authoritative DNS |
| `10.10.10.20` | `client.lab.test` | Untrusted sender, test tooling, private CA |
| `10.10.10.21` | — (no PTR) | Client alias: SPF-unauthorized source for negative tests |

Static addressing via NetworkManager keyfiles. Guests resolve against `10.10.10.10`.

**Mail VM as built:** `ens160` manual `10.10.10.10/24`, no gateway, no DNS server yet (BIND
arrives in Section 3), IPv6 disabled. Routing table holds only the connected route
`10.10.10.0/24 dev ens160`; there is **no default route** and no IPv6 route. `ens192` is down
with no carrier.

**Client VM as built:** `ens160` manual `10.10.10.20/24` (primary) and `10.10.10.21/24`
(kernel-flagged `secondary`), no gateway, no DNS server yet, IPv6 disabled
(`disable_ipv6 = 1` on `ens160`). The routing table holds only the connected
`10.10.10.0/24` routes, one per source address; there is **no default route** and no IPv6
route. `ip route get 10.10.10.10` selects source `10.10.10.20`, so traffic leaves from `.20`
unless a test binds `.21` explicitly. `ens192` is down with no carrier.

## Management access

| Item | Recorded state |
|---|---|
| Admin account | `labadmin`, member of `wheel` (D8, D14); root account disabled |
| SSH client | Windows OpenSSH on the host, run from PowerShell or from WSL as `ssh.exe` (D10). WSL's own `ssh` is not used for the lab: in WSL NAT mode it would depend on the host forwarding into VMnet10, which B1 forbids. |
| Console | Break-glass and out-of-band checks only |
| Host key | Each VM's ED25519 host key fingerprint was read at the console and matched before the first SSH connection was trusted (mail VM in 2.3, client VM in 2.4) |
| Authentication | Password, temporarily; key-only login and `PermitRootLogin no` are configured on both VMs in 2.5 |
| Evidence transfer | Captured on the VM, copied to the host with Windows OpenSSH `scp` (from PowerShell, or `scp.exe` from WSL), and SHA-256 compared on both sides. Piping `ssh.exe` output inside WSL stalls at an invisible password prompt and is not used. |

## Time

| Item | Recorded state |
|---|---|
| Time zone | `Etc/UTC` (D13) on both VMs; RTC in UTC |
| chronyd | Installed, inactive on both VMs (installer "Network Time" off) |
| VMware Tools time sync | Off in both VMs' settings; the client has no VMware Tools installed at all |

**F-01:** The installer clock was eight hours behind UTC on both the mail VM and client VM,
and was manually corrected before installation. The client's first installed-system boot and
subsequent cold boot showed correct UTC despite `open-vm-tools` being absent. The underlying
cause and recurrence conditions remain unconfirmed. Investigation and the D6
time-synchronization revision are assigned to Stage 2.5.

**F-02:** while the host slept overnight the running VM's clock stopped, leaving it about five
hours behind. With Tools time sync off and no reachable time source, nothing corrects it.
This invalidates the D6 assumption that chrony only needs a source during maintenance
windows; a revised D6 is proposed in 2.5, before Section 4 depends on accurate time.

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
| AlmaLinux 10 BaseOS / AppStream | OS, Postfix, Dovecot, BIND, MariaDB, httpd, PHP, Valkey | `EE6D B7B9 8F5B F5ED D9DA 0DE5 DEE5 C11C C2A1 E572`, matched to almalinux.org; it signed the 10.3 ISO checksums in 2.2 |
| EPEL 10 | Roundcube, fail2ban, swaks, python3-dkimpy | *recorded in Section 2 (first maintenance window)* |
| rspamd upstream (EL10) | rspamd | *recorded in Section 9* |

`gpgcheck=1` everywhere. No `--nogpgcheck`.

## Installed versions

| Component | Version | Recorded in |
|---|---|---|
| AlmaLinux | **10.3 (Mauve Lion)**, kernel `6.12.0-264.16.1.el10_3` on both VMs — 10.3 instead of the locked 10.2 (D11) | Section 2 (2.3, 2.4) |
| BIND | | Section 3 |
| MariaDB | | Section 5 |
| Postfix | | Section 6 |
| Dovecot (must be 2.3.x) | | Section 7 |
| rspamd | | Section 9 |
| Valkey | | Section 9 |
| httpd / PHP | | Section 10 |
| Roundcube | | Section 10 |
| fail2ban | | Section 11 |

## Section 2 decisions and deviations

Owner decisions taken during Section 2. They are submitted for the Reviewer's ruling in
S02-R04 and do not reopen Section 1.

| # | Decision |
|---|---|
| D6 | Mail VM is the lab time authority; **revision due in 2.5** after F-02 |
| D7 | Default LVM/XFS with 10 GiB of the mail VM's volume group left free for `/var/vmail` |
| D8 | Non-root `wheel` admin, SSH keys, key-only login, root login disabled |
| D9 | VMs stored on the host's data drive |
| D10 | Administration from the host via Windows OpenSSH (PowerShell, or WSL through `ssh.exe`) |
| D11 | **Deviation:** AlmaLinux 10.3 installed (released 2026-11-16), not the locked 10.2 |
| D12 | **Deviation:** plain UEFI; VMware offered no Secure Boot option |
| D13 | Time zone UTC |
| D14 | Admin account `labadmin`, keeping personal names out of evidence |

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
backup-and-restore process is Section 13. No Section 2 snapshot has been taken yet; it is
taken at the end of the section.

## Evidence

Raw evidence — terminal transcripts, command output, screenshots — is held outside the
repository. Screenshots are not committed while image evidence awaits a Reviewer decision.
Verification captures are retained offline with these SHA-256 values:

| Capture | SHA-256 |
|---|---|
| Mail VM first verification (2.3c) | `8239b6b6e476c7f0852d1a136e035acf23c18811acc5579e2f781b0d08a0168a` |
| Client VM first verification (2.4c) | `bb17b96aaa339c18dc3f2be1d067661bf8f4e85266ed55febb0410111f26136b` |
| Client VM supplement: optical media, source address (2.4c) | `407280bd641408b57dac43fe6c7fb99e306a84b61ca82dd233b0dd50b3855dc1` |
