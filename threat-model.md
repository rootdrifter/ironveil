# ironveil — Threat Model

A structured threat model for the ironveil hardened workstation. The [README](README.md) documents
*what* controls are in place; this document asks the harder questions: **who is the realistic
adversary, what is actually being protected, and for each way in — what stops it, what residual risk
remains, and where the honest gaps are.** A threat model with no gaps is not a threat model; the gaps
below are deliberate and tracked.

> Method: a defined adversary, an explicit asset inventory, then per-vector analysis
> (vector → control → residual risk → honest gap). Device-wide reasoning and a residual-risk
> statement close it out. Specific keys, credentials, and the origin/host network values are out of
> scope by policy.

## 1. Adversary model

The realistic adversary for a security-cleared practitioner's personal workstation is **not** a
nation-state with an unlimited 0-day budget. Modelling against that adversary is both undefendable and
dishonest. The credible threat is a **resourceful, targeted attacker who knows this machine belongs to
someone with clearance and sensitive material**, and who has one or more of: physical access through
theft or seizure, a position on the network, or a way to phish a credential.

| Adversary | Capability | Primary goal | In scope? |
|-----------|-----------|--------------|-----------|
| Opportunistic thief | Physical possession of a powered-off machine | Resale; casual data access | ✅ |
| Targeted physical adversary | Theft/seizure knowing the target; can image the disk and attempt offline derivation | Clearance evidence, research, credentials | ✅ |
| Network adversary (hostile Wi-Fi, rogue AP, on-path) | Traffic interception, DNS manipulation, packet injection | Surveillance, credential capture, geolocation | ✅ |
| Targeted social-engineering / phishing | Trick the user into authorising or disclosing | Credential theft, foothold | ✅ |
| Evil-maid (brief physical access, device returned) | Tamper with firmware/bootloader, leave and wait | Persistent pre-OS implant | ◐ (partial — see gaps) |
| Nation-state full-chain | Chained 0-days, supply-chain implants, unlimited resources | Total compromise | ⬜ (out of realistic scope — acknowledged) |

**Trust anchor (and its honest limit).** ironveil's unlock-integrity guarantee rests on
**LUKS2 + touch-only FIDO2 + a pinned initramfs SSH host key** — *not* on a hardware-measured boot
chain. UEFI Secure Boot is currently disabled (Setup Mode) and the LUKS unlock is not sealed to TPM2
PCR measurements. This means the trust anchor is strong against the **powered-off disk-theft**
adversary (the disk does not open without a hardware key or the offline passphrase) but **weaker
against the evil-maid adversary** (see V1/V5). This is stated up front because it is the model's most
important boundary.

## 2. Assets — what is actually being protected

| Asset | Why it matters | Sensitivity |
|-------|----------------|-------------|
| Clearance status & integrity | No event on this machine may become evidence of a security lapse | Highest |
| Application / identity material | CV, personal details, correspondence (kept off the repos by policy) | High |
| Unpublished security research | Findings, engagement notes, tooling not yet (or never) public | High |
| Access credentials | GitHub (push as `rootdrifter`), Ghost admin, cloud/VPS, SSH keys | Highest |
| Professional communications | Email, contacts, in-flight conversations | Medium-High |

The asset inventory drives the controls: the highest-sensitivity assets (clearance integrity, access
credentials) are exactly what the FIDO2-gated unlock and the credential-custody design protect.

## 3. Per-vector analysis

For each vector: the control that mitigates it · the residual risk that remains · the honest gap.

### V1 — Physical access: offline disk clone & brute-force
- **Control:** LUKS2 `aes-xts-plain64` / 512-bit, **Argon2id** passphrase slot (memory-hard), unlock
  gated on a touch-only Nitrokey 3A NFC FIDO2 token. The disk does not open without a hardware key or
  the offline emergency passphrase; Argon2id makes offline guessing impractical to parallelise on
  GPU/ASIC.
- **Residual:** the **cold-boot / DMA-while-running** class — LUKS keys live in RAM while the machine
  is unlocked and running; an attacker who seizes it *powered-on and unlocked* is a different problem
  than one who seizes it off.
- **Honest gap:** no RAM-wipe-on-power-loss and no measured-boot binding; mitigation today is
  operational (lock/suspend-to-disk discipline, power off when unattended). Tracked as future work
  alongside TPM2.

### V2 — Primary hardware key lost or seized
- **Control:** key custody is deliberately split across three LUKS keyslots — primary Nitrokey (daily),
  **backup Nitrokey kept offline**, and an **offline emergency passphrase**. Losing the primary key is
  a recoverable inconvenience, not a lockout.
- **Residual:** the offline passphrase is a single high-value secret; if it and a token were ever
  captured together, that is game over.
- **Honest gap:** none structurally — but the passphrase's offline storage is a process control, not a
  technical one, so it is only as strong as the operator's handling of it.

### V3 — Remote unlock path (dracut-sshd)
- **Control:** the initramfs runs a minimal SSH daemon (`dracut-sshd` v0.7.1-5.fc44, systemd-networkd +
  fido2 modules) reachable only from the trusted GrapheneOS handset over Tailscale; the initramfs SSH
  **host key is pinned on the client**, and the LUKS secret is **never sent over the network** — a
  Nitrokey touch at the machine satisfies the prompt, relayed by `systemd-tty-ask-password-agent`.
  Completing an unlock requires the physical handset **and** its Termux key file.
- **Residual:** the pre-boot network surface exists while the machine waits at the unlock prompt;
  reachability depends on the Tailscale tunnel and the LAN the pre-boot interface sits on.
- **Honest gap:** the pre-boot SSH daemon is a (small, key-only, pinned) attack surface that a
  fully-offline machine would not have — an accepted trade for headless remote unlock.

### V4 — Network interception, geolocation, DNS leakage
- **Control:** WireGuard full-tunnel (`wg-CH-FI-2` / `wg-SE-FI-1`, `AllowedIPs 0.0.0.0/0, ::/0`) carries
  all external traffic; AdGuard Home on `*:53` forwards **only** to Quad9 over DNS-over-HTTPS, so every
  query leaves already encrypted and egresses through the active tunnel. An on-path observer sees only
  WireGuard ciphertext; the upstream resolver sees the tunnel exit IP, never the host, and never a
  plaintext query.
- **Residual:** **VPN-provider trust** — the exit operator can see post-tunnel metadata; and a tunnel
  drop is only *implicitly* fail-closed.
- **Honest gap:** the kill-switch is **route-based, not a fail-closed firewall rule** — while a tunnel
  is up it is the default route, but there is no nftables rule that hard-drops traffic if the interface
  disappears. A hard kill-switch is documented as planned, not implemented.

### V5 — Evil-maid / boot-chain tamper
- **Control:** the pinned initramfs host key means an attacker cannot transparently MITM the unlock;
  touch-only FIDO2 means a swapped key cannot be activated remotely.
- **Residual:** firmware/bootloader tamper that does **not** touch the pinned host key is not detected
  by the current build.
- **Honest gap:** **UEFI Secure Boot is disabled (Setup Mode)** and the unlock is **not sealed to TPM2
  PCRs**, so a measured-boot evil-maid implant below the OS is the model's weakest point. Closing it
  (Secure Boot enrolment + TPM2 PCR sealing) is the top tracked future-work item.

### V6 — Credential compromise (phishing, software keystore theft)
- **Control:** Nitrokey FIDO2 is the second factor for the services that support it; SSH push auth uses
  a hardware-held key; vendor cloud daemons (Synapse/iCUE) that phone home are eliminated, reducing the
  software that holds or transmits secrets.
- **Residual:** services that **do not** support FIDO2 fall back to weaker factors.
- **Honest gap:** the FIDO2-everywhere goal is bounded by provider support; that subset is an accepted,
  documented residual.

### V7 — Post-exploitation / privilege escalation (a foothold already exists)
- **Control:** **SELinux enforcing** (`targeted` policy) provides mandatory access control above DAC;
  **seccomp BPF** (`CONFIG_SECCOMP_FILTER=y`) sandboxes systemd services; minimal service footprint
  limits what a foothold can reach.
- **Residual:** kernel 0-day; a `targeted` policy is not a confined-everything `strict`/MLS policy.
- **Honest gap:** **Yama `ptrace_scope` is at the Fedora default `0`** (tightening to `1` planned), so
  same-uid process introspection is broader than it needs to be.

## 4. Device-wide residual risk — the honest statement

ironveil is **strong against the most probable adversary**: a powered-off stolen or seized machine
yields an encrypted disk that does not open without hardware-key presence or the offline passphrase,
and all network traffic is tunnelled and DNS-encrypted. It is **deliberately honest about its weakest
edges**: the evil-maid / measured-boot gap (no Secure Boot, no TPM2 PCR sealing), the cold-boot class
while running, the route-based rather than fail-closed kill-switch, and VPN-provider trust. None of
these are hidden — each is tracked, and the highest-value one (measured boot) is the named next build.
That is the point of the document: the value of a workstation build is not a claim of perfect security,
it is the demonstrated discipline of knowing exactly where the boundary is.

---

*Part of the [rootdrifter](https://github.com/rootdrifter) security portfolio. Companion to the
nullbyte [threat model](https://github.com/rootdrifter/nullbyte/blob/main/threat-model.md) — the
workstation and the handset modelled against the same adversary set.*
