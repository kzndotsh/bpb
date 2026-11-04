# Architecture

## Platform by role

The workload picks the box's OS, not hardening ambition. Kernel hardening raises the cost of rungs 1 and 2. Rungs 4 to 6 (hypervisor, BMC, legal) sit below or outside any guest kernel, and the design already concedes root to the host from second zero. bpb's value is in off-box evidence, ephemerality, Tang and the airlock, so the OS for each role is chosen for what that role has to do.

| Role | OS | Why |
|---|---|---|
| Box | NixOS, built as a dm-verity image | CUDA, the NVIDIA driver and the ML stack need Linux with glibc. `nixos-anywhere`, `disko` and `nixosTest` have no equivalent elsewhere. The closure hash is what attestation compares against |
| Anchor | NixOS | It builds every closure and runs the independent-rebuild comparison. Plain NixOS with FDE |
| Witness | NixOS first, OpenBSD as a later milestone (M11) | No GPU and no ML stack. Its main risk is parsing hostile telemetry, which pledge/unveil fits well. A different kernel from the box means one kernel bug cannot take both the box and its watcher |
| Relay | NixOS first, OpenBSD as a later milestone (M11) | pf and wg(4) are in OpenBSD's base system. It does nothing else |

Witness and relay start on NixOS so every non-hardware test runs under `nixosTest` through M10. `nixosTest` only builds NixOS VMs, so testing an OpenBSD witness means booting an OpenBSD image in QEMU from a custom harness. That cost is why it waits.

**Why not the alternatives for the box:**

- **OpenBSD, FreeBSD:** no supported CUDA, so the job cannot run. They also lack what the plan is built on: LSM-hook or eBPF sensors, network-bound unlock (Tang/Clevis), kexec install, a reproducible closure, a VM-test framework, mature SEV-SNP/TDX guest support. Their strengths (pledge/unveil, KARL, Capsicum, securelevel, DTrace) favor prevention; bpb is mainly a detection project.
- **musl:** manylinux wheels and the CUDA userspace assume glibc. Use `pkgsMusl` or static builds for individual bpb services where it fits, not the whole box.
- **Gentoo hardened:** the same Linux kernel, minus the reproducible closure. NixOS already gives the parts Gentoo would be used for:
  - a stripped kernel built declaratively (`boot.kernelPatches`, structured config) or `linux_hardened`
  - a hardened allocator system-wide (`environment.memoryAllocator.provider = "graphene-hardened"`, or `scudo`)

**The box as a verity image.** By default `/nix/store` is a writable filesystem: root can modify it, and `nix store verify` running on the box is the box's own claim. Instead, the box is built with nixpkgs' `repart-verity-store` module (`image.repart.verityStore.enable`)[^repart]. It produces a read-only, dm-verity-protected store partition whose root hash goes on the kernel command line of a unified kernel image (UKI).

- A tampered store page fails verification when read, not at the next periodic check.
- On metal with a TPM, measured boot now covers the whole system: the TPM's PCRs record the UKI, and the UKI carries the store's root hash, so a quote proves which OS booted. Attestation stops being self-reported. The quote must include PCR 12 as well as PCR 11, because without Secure Boot the UKI's embedded command line (and with it the root hash) can be overridden at launch, and PCR 12 is where that override shows up (see Network). Secure Boot is on wherever the tier allows it: on NixOS it is self-signed (Lanzaboote, still out of tree)[^lanzaboote], and upstream's own primary target is measured boot, not Secure Boot.[^bootissue]
- **Deploying the image.** `nixos-anywhere` normally installs a closure into partitions that `disko` creates, while `repart-verity-store` produces a whole disk image that needs the systemd initrd. So the box path is: `nixos-anywhere --phases kexec` to reach the RAM installer, then write the image to disk directly (`dd`, or `systemd-repart` from the image's definitions), and create the LUKS work partition separately at arm. M6 proves this path end to end; until then, earlier milestones use a plain NixOS box.
- Encrypting the OS instead would not give this. Attestation could then prove only that the encrypted disk booted, whatever it contains, not that a trusted OS did.
- Root stays on tmpfs. The impermanence model is unchanged; only the store becomes immutable.
- The cost is that a verity box is not normal NixOS: there is no `nixos-rebuild`, and any change means building and writing a whole new image. A NixOS maintainer calls these images "reliable and worth using"[^thread] with exactly that caveat. bpb boxes are never updated in place (they are burned and replaced), so the cost does not apply. Upstream plans a measured, mutable store ("stage 2 measurement", #549372), which is worth revisiting for the anchor once it exists.

**systemd.** NixOS depends on it, and bpb now relies on it for real controls: unit sandboxing, `systemd-cryptenroll`, `systemd-repart`, UKIs, verity in the initrd, TPM measured boot. Dropping it would mean rebuilding each of those by hand. Its size is managed by turning off what bpb does not use (see Integrity and supply chain).

**Considered and rejected for the box: sixos.** sixos is a nixpkgs-based OS that replaces systemd with skarnet's s6 supervision suite. It shares no code with NixOS, and it composes services as package-like Nix expressions through `infuse.nix` instead of the NixOS module system.[^sixostalk] It is active (commits through October 2026) and has a single maintainer.[^sixosrepo] s6 is far smaller than systemd, so the attack-surface argument is real. It is also small for bpb: the box concedes root to the host from second zero, so systemd's size matters mainly as a privilege-escalation path for rung 1, which the trimmed component list already narrows. Against that, sixos would cost:

- the systemd controls listed above
- `nixos-anywhere`, `disko` and `nixosTest`, which all target NixOS. `nixosTest` is what makes "a control without a firing test does not count" practical.
- the NVIDIA stack, which nixpkgs wires up through NixOS modules (whether sixos can run it is unchecked)
- a maintainer base of more than one person

Community reaction was mixed: most interest was in `infuse.nix`, and a NixOS contributor called several of the talk's systemd claims dishonest.[^sixosthread] Finix, a finit-based alternative shown at NixCon 2025, is in the same space and has not been looked at.

## Module map

```
flake.nix
modules/
  box/
    base.nix         bpb's own kernel config (nixpkgs' hardened profile is gone), sysctls,
                     LSM list, no-swap or random-key swap, core dumps off (piped),
                     hardened allocator, unused systemd components off,
                     minimal + perlless profiles, anchored forbidden-dependency deny list
    disk.nix         verity store image (repart) + UKI, tmpfs root, read-only /etc overlay,
                     LUKS work volume formatted at arm from an anchor-generated key
    unlock.nix       bpb's own unlock client: Tang recovery + anchor-held second factor
    bootstrap.nix    enrollment credential, attestation + event keys, nonce-bound quote,
                     sealed delivery of runtime keys
    impermanence.nix erase-your-darlings, explicit /persist allowlist (empty by default)
    net.nix          nftables default-deny in AND out; WG + Rosenpass to the relay
                     (job traffic, tunnelled to the anchor) and to the witness (evidence)
    dns.nix          unbound, allowlist forwarders, NXDOMAIN everything else
    access.nix       sshd on wg0 only, short-TTL SSH certs, no passwords, no agent/TCP/
                     stream-local/X11 forwarding, pinned PQ KEX/AEAD ciphers/ETM MACs,
                     anchor-signed host + user certificates only, compression off,
                     listener inventory, no serial/console getty, root locked,
                     no sudo, no default packages, no Nix
    control.nix      bpb-control principal: force-command to a root socket service
                     with fixed verbs (arm, burn-local, redteam-case)
    job.nix          the job's sandboxed unit, resource limits, optional JIT cache mount
    sensors.nix      auditd + BPF-LSM/tetragon rules, buffered shipper with resend,
                     bpb's own journald forwarder → witness, nonce-bound heartbeat
    canaries.nix     honeyfiles, per-box honey creds, DNS canaries, honey user,
                     honey port (nftables log-and-drop)
    attest.nix       verity root hash + closure hash report, TPM/SNP quote when present
    lockdown.nix     late-boot one-way switches (modules, kexec if present, bpf, lockdown)
    burn.nix         cryptoshred unit, run through the control service
  anchor/
    base.nix         FDE unlocked by two enrolled tokens + PIN, one key per role,
                     same lockdown and sensors as the box, per-peer nftables,
                     mutual heartbeat with the witness, WG to the relay only
    keystore.nix     per-run keystore (Tang key, unlock secret, mapping, run_key),
                     envelope-encrypted, excluded from every backup
    tang.nix         one tangd instance per box on its own port, firewalled to that box,
                     enabled only at arm against a witness receipt
    resolver.nix     unbound for boxes: DNSSEC validation, DNS-over-TLS upstream
    rosenpass.nix    post-quantum PSK feed for every WG peer, initial PSK at delivery
    airlock.nix      (microVM) export sanitizer, return-path validator, known-answer checks
    ledger.nix       hash chain over salted commitments, signed with the ledger key
    collector.nix    (microVM) receives job traffic through the hostile-input ingest
    backup.nix       age, multi-recipient, 3-2-1, off-site copy holds pruned exports only
  relay/
    forward.nix      UDP forward of box WG + Rosenpass to the anchor, egress for
                     provisioning, nothing else
  witness/
    sink.nix         append-only receiver (raw bytes first, sandboxed strict parser),
                     own retention clock, never deletes commitments
    verdicts.nix     correlation, signed verdicts and "stream healthy" receipts to the
                     anchor, heartbeat watchdog for boxes and anchor
    alerts.nix       self-hosted ntfy, no Apple/Google push, scheduled end-to-end self-test
builder/
  rebuild.nix       independent second builder for the box closure, substituters off,
                    reports NAR hashes
cli/
  bpb               provision | attest | arm | run | drain | freeze | unfreeze | resume |
                    burn | verify-burn | redteam | hold | unhold
tests/
  *.nix             nixosTest VM tests; [hw] tests run on real hardware or a provider
```

## Box lifecycle as a state machine

```
provisioned → attested → armed → running → draining → burned → verified-burned
      │            │         │        │  ▲       │
      │            │         │        ▼  │ (bpb unfreeze, same box still up)
      └────────────┴─────────┴──► frozen ┴───────┴──► burned   (human command only)
                                     │
                                     └──► resumed on a fresh box (new keys, last checkpoint)
```

**No secret reaches the box before it has proved what it is.** Earlier drafts injected keys at install time, through a first-use SSH session, before attestation; that let the host capture them, and keys written at install time land on disk, which contradicts "lost on reboot". The order is now:

- **provisioned.** The provider API calls and the `nixos-anywhere` session leave through the relay, so the provider's logs show the relay, not the anchor. The RAM installer's SSH host key is generated on the anchor and handed to the provider as user-data or through the provider API, so the anchor pins it out of band instead of trusting it on first use. `nixos-anywhere --phases kexec` boots that installer, which writes the box's verity disk image (a UKI whose command line carries the store's root hash) and a single-use **enrollment credential**: a WireGuard key the anchor will accept exactly once, from this box's run ID, for attestation only. The image itself contains no per-box secret, so every box of a closure has the same image hash. The box reboots into the image.
- **attested.** On boot the box generates an ephemeral attestation key pair and an event-signing key pair, connects with the enrollment credential, and asks the anchor for a nonce.
  - **TPM (metal):** a quote over the measured-boot PCRs, including PCR 11 (the UKI) and PCR 12 (any command-line override, see Network), with the nonce and a hash of both public keys in the qualifying data. The anchor verifies the TPM's endorsement-key certificate chain to the manufacturer and rejects a quote from a virtual TPM, which only vouches for the host.
  - **SEV-SNP/TDX:** a report with the 64-byte `REPORT_DATA`/`REPORTDATA` field set to H(nonce ‖ attestation key ‖ event key), which binds the report to this session and these keys.[^snpabi][^tdxattest] The anchor verifies:
    - the signature and certificate chain (SNP: VCEK ← AMD SEV CA ← AMD root, fetched from AMD's KDS; TDX: the PCK chain to Intel's SGX root, with collateral from Intel's PCS)
    - the launch measurement equals the bpb UKI the anchor built: SNP's launch digest, or TDX's `MRTD` plus the expected `RTMR` values. This is why kexec is ruled out on T4 (see Escalation tiers): kexec runs a kernel the launch never measured.[^kexecsnp]
    - the **debug bit is off** (SNP policy `DEBUG` clear; TDX `DEBUG` attribute clear). A debug confidential VM gives the host read/write access to its memory and no confidentiality at all.[^tdxattest]
    - the **reported TCB is at or above a pinned minimum**, including the fix for RMPocalypse (CVE-2025-0033), which until mid-2025 let a malicious host forge SNP attestation reports outright. The `REPORTED_TCB` is bound to the VCEK, so a host below the fixed microcode/firmware level cannot present a report that passes this check.[^rmpocalypse] TDX gets the equivalent minimum SEAM/TCB floor. These minimums are reviewed as new confidential-computing breaks land, which is often.
  - **No hardware:** the box reports its closure hash and public keys. This catches drift and sloppy tampering, not rung 2+, and the ledger records it as `"kind": "none"`: not attested, never a weaker pass. Key delivery to such a box is trust-on-first-use, and the ledger says so.

  Why only hardware counts: whoever controls a server can change what it runs while it keeps looking normal. Dutch police ran Hansa Market for a month after taking its servers over, and modified its code to capture passwords and order details before they were encrypted.[^hansa] A self-reported hash from such a box would have said nothing was wrong. On success the anchor revokes the enrollment credential, registers the box's event key with the witness, and **seals** the box's runtime keys to the attested key (TPM `MakeCredential`/`ActivateCredential`, or encryption to the SNP-bound key): the runtime WireGuard key, the initial Rosenpass PSK, the SSH host key and its anchor-signed host certificate. The anchor deletes its copies of every private half immediately after delivery; it only ever needs the public halves.
- **armed.** Late-boot one-way switches flip (see Lockdown). Canaries are laid out with a fresh random layout. The witness confirms the evidence stream is flowing and sends the anchor a signed "stream healthy" receipt. Only against that receipt does the anchor format the work volume with an anchor-generated volume key (see Disk and memory) and enable this box's Tang instance.
- **running.** The job runs. Job traffic goes only to the anchor (through the relay); evidence goes only to the witness (directly). The witness gets events and a heartbeat every 10 seconds. The anchor pulls a checkpoint through the airlock at a job-defined interval, so a false trip costs minutes of work, not the whole job.
- **draining.** Outputs are pulled over WG and validated in the airlock quarantine. The anchor drops the box's WG peer only after burn step 3 has run on the box.
- **frozen.** The anchor stops sending data, pauses checkpoints at the last good one, and disables the box's Tang instance (moved out of the served set; the key still exists). Freeze is reversible in two ways. If the box is still up and the operator clears the signal, `bpb unfreeze` returns it to `running`. If the box has rebooted or powered off, it cannot rejoin: its runtime keys lived only in RAM, its enrollment credential is revoked, and the anchor never re-delivers keys to a box that has already been delivered to. "Resume" then means a fresh box with fresh keys, restarting from the last checkpoint. Freeze keeps the data; it does not always keep the job.
- **burned.** Cryptoshred first, provider API destroy second. See Burn. Burn is never automatic: it is `bpb burn`, run by a human (see Response ladder).
- **verified-burned.** `bpb verify-burn` passes: this box's Tang instance and keys are gone from the anchor, and the provider's API lists zero instances, volumes, snapshots, images and backups for it (see Burn). The ledger entry closes.

Every transition is idempotent. Re-running `bpb burn` on a half-burned box converges to `verified-burned`.

---

[^repart]: nixpkgs, `nixos/modules/image/repart-verity-store.nix`. https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/image/repart-verity-store.nix

[^lanzaboote]: nix-community, Lanzaboote (Secure Boot for NixOS). https://github.com/nix-community/lanzaboote

[^bootissue]: nixpkgs issue #549372, "End-to-end NixOS boot security" (tracking; opened August 2026; autopen, deterministic module signatures #571181, stage 2 measurement). https://github.com/NixOS/nixpkgs/issues/549372

[^thread]: r/NixOS, "NixOS for high threat model server" (May 2025), comments by NixOS maintainer ElvishJerricco on dm-verity images, module signing, lockdown, MAC and the hardened profile, and on microvm.nix. Read via Arctic Shift. https://www.reddit.com/r/NixOS/comments/1kvpezv/

[^sixostalk]: Adam Joseph, "sixos: a nix os without systemd", 38C3 (December 2024). https://app.media.ccc.de/v/38c3-sixos-a-nix-os-without-systemd and https://pretalx.c3voc.de/38c3/talk/690/

[^sixosrepo]: amjoseph, sixos repository on Codeberg (last commit checked: October 28, 2026). https://codeberg.org/amjoseph/sixos

[^sixosthread]: NixOS Discourse, "sixos: a nix os without systemd" (January–September 2025, 26 posts). https://discourse.nixos.org/t/sixos-a-nix-os-without-systemd/58141

[^snpabi]: AMD, "SEV Secure Nested Paging Firmware ABI Specification" (rev. 1.58, May 2025): the 64-byte `REPORT_DATA`, `REPORTED_TCB`, and the VCEK-signed attestation report. https://www.amd.com/content/dam/amd/en/documents/developer/56860.pdf — and Asad & McClurg, "AMD SEV-SNP: A Confidential Computing Primer" (arXiv 2608.04039), for the verifier flow, plus the `virtee/snpguest` reference verifier. https://arxiv.org/pdf/2608.04039 and https://github.com/virtee/snpguest

[^tdxattest]: Cheng et al., "Intel TDX Demystified" (arXiv 2303.15540): TD quote generation, `MRTD`/`RTMR`/`REPORTDATA`, the PCK chain to Intel's root, and the requirement that the TD `DEBUG` attribute be clear. https://arxiv.org/pdf/2303.15540

[^kexecsnp]: kexec mailing list, "x86/snp: Add kexec support" (May 2024): the launch measurement indicates the directly booted environment, and a kexec'd kernel runs an environment the launch never measured. https://lists.infradead.org/pipermail/kexec/2024-May/029990.html

[^rmpocalypse]: RMPocalypse (CCS 2025), demonstrating SEV-SNP attestation forgery, code injection and register-state replay. https://rmpocalypse.github.io/rmpocalypse-CCS2025.pdf

[^hansa]: Help Net Security, "Dark web Hansa Market shut down after being run for a month by law enforcement" (July 2017). https://www.helpnetsecurity.com/2017/07/20/hansa-market-shut-down/
