# LUKS2 Full-Disk Encryption Setup

Living build — hardware values captured from the running machine on 2026-06-11. `FUTURE WORK`
markers flag planned enhancements; any remaining `MANUAL INPUT REQUIRED` marker is a value not yet
captured.

## Encrypted volume

| Attribute | Value |
|-----------|-------|
| LUKS container UUID | `6cbc50ba-6f8a-4932-abfc-f2d0504a29b3` |
| Mapped device | `luks-6cbc50ba-6f8a-4932-abfc-f2d0504a29b3` |
| Format | LUKS2 |
| Cipher | `aes-xts-plain64`, 512-bit key |
| Key derivation (keyslot 0) | Argon2id (passphrase slot) |
| Key derivation (keyslots 1–2) | PBKDF2/SHA-512 (FIDO2 slots) |

The volume is referenced by UUID throughout (crypttab and the kernel command line both use
`rd.luks.uuid=luks-6cbc50ba-…`), so commands below use `/dev/disk/by-uuid/6cbc50ba-…` rather than a
fixed `/dev/sdX` node.

Argon2id (the passphrase slot) is memory-hard, making offline brute-force of a cloned disk
impractical without the hardware key or the emergency passphrase. The FIDO2 slots derive their key
material via PBKDF2/SHA-512 in combination with the Nitrokey credential.

## Keyslot layout

Three keyslots are active, providing redundancy without weakening the security model:

| Slot | Type | KDF | Role |
|------|------|-----|------|
| 0 | Passphrase | Argon2id | Emergency / recovery fallback — long, high-entropy, stored offline |
| 1 | Nitrokey 3A NFC (primary) | PBKDF2/SHA-512 | Daily driver — physical touch required for every unlock |
| 2 | Nitrokey 3A NFC (backup) | PBKDF2/SHA-512 | Kept offline; activated only if the primary is lost |

Both FIDO2 slots are enrolled with `fido2-up-required=true` (a physical touch is mandatory) and
`fido2-clientPin-required=false` (no PIN — consistent with the Nitrokey 3A NFC's touch-only
capability; user verification is not required). Two FIDO2 tokens are enrolled.

Inspect the header with:

```
sudo cryptsetup luksDump /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3
```

## FIDO2 enrolment

The hardware keyslots are enrolled with `systemd-cryptenroll` using FIDO2. The LUKS2 passphrase
is never transmitted to the key; the Nitrokey produces a FIDO2 credential combined with a stored
key file to derive the slot key.

```
# Primary hardware token — enrolled while the primary key is inserted
sudo systemd-cryptenroll /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3 --fido2-device=auto

# Backup hardware token — enrolled while the backup key is inserted
sudo systemd-cryptenroll /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3 --fido2-device=auto
```

**Touch-only enrolment:** the Nitrokey 3A NFC on firmware 1.8.3 does **not** support
`clientPin`. Both tokens are therefore enrolled with `fido2-clientPin-required=false` and
`fido2-up-required=true` — a physical touch is mandatory for every unlock, but no PIN is involved.
This is intentional — a PIN-over-USB credential could be activated without physical presence,
defeating the presence guarantee. See [nitrokey.md](nitrokey.md) for the hardware-key detail.

```
# clientPin is not enrolled; presence is enforced by touch (user-presence required)
sudo systemd-cryptenroll /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3 \
  --fido2-device=auto --fido2-with-client-pin=no
```

## crypttab configuration

`/etc/crypttab` references the volume by UUID and instructs systemd to use the FIDO2 device, with
the passphrase slot (slot 0) as the fallback if no token is present. The live entry:

```
# <name>                                          <device>                                       <keyfile>  <options>
luks-6cbc50ba-6f8a-4932-abfc-f2d0504a29b3  UUID=6cbc50ba-6f8a-4932-abfc-f2d0504a29b3  none       discard,x-initrd.attach,fido2-device=auto
```

Option notes:

- `fido2-device=auto` — FIDO2 unlock is the live mechanism (touch a Nitrokey at the prompt); it is
  not a passphrase-only setup.
- `discard` — SSD TRIM is enabled on the mapped volume. This is the deliberate SSD/threat trade-off:
  TRIM can reveal which blocks are unused (a minor metadata leak) in exchange for sustained SSD
  performance and lifespan.
- `x-initrd.attach` — the volume is attached in the initramfs, which is what allows the pre-boot
  FIDO2 / dracut-sshd unlock path (see [dracut-sshd.md](dracut-sshd.md)).

## Verification

```
sudo cryptsetup luksDump /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3   # 3 active keyslots
sudo systemd-cryptenroll /dev/disk/by-uuid/6cbc50ba-6f8a-4932-abfc-f2d0504a29b3 --fido2-device=list
lsblk -f                                                                           # confirm the mapped volume
```

## Current state

| Item | Status |
|------|--------|
| LUKS2 `aes-xts-plain64`/512-bit on `luks-6cbc50ba-…` | Operational |
| Slot 0 — emergency passphrase (Argon2id) | Active; stored offline |
| Slot 1 — primary Nitrokey 3A NFC FIDO2 (touch-only) | Operational |
| Slot 2 — backup Nitrokey 3A NFC FIDO2 | Enrolled; stored offline |
| Build platform | Fedora release 44 (Forty Four), kernel `7.0.11-200.fc44.x86_64` |

<!-- MANUAL INPUT REQUIRED: benchmark unlock latency (hardware key vs passphrase) — time each unlock path on the running machine and record the figures. Not captured in the 2026-06-11 hardware session. -->
<!-- FUTURE WORK: record the LUKS2 verified-boot / Secure Boot state (distinct from the LUKS UUID) once captured — `mokutil --sb-state` and the measured-boot configuration. -->

> **Note on the FIDO2 credential:** the per-token credential IDs reported by `luksDump` are hardware
> attestation data and are intentionally **not** reproduced here — only the token count (2) and the
> enrolment policy (touch-required, no clientPin) are documented.
