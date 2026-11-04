## Milestones

Each ends with a passing test. The order builds a thin end-to-end slice first, so a real job runs early, and then hardens it. Earlier drafts front-loaded everything into M0 and ran nothing real until M8.

Tests marked **[hw]** need real hardware or a real provider (a GPU, a physical TPM over a real firmware boot, a provider console or firewall). Everything else runs in `nixosTest` VMs in CI. Tests that need network access outside the Nix build sandbox are marked **[net]** and run in a separate CI job.

1. **M0. Thin slice.** A plain NixOS box (no verity yet) installed by `nixos-anywhere` + `disko` into a local QEMU VM, with anchor, relay and witness VMs. Job-traffic WireGuard via the relay; a separate evidence peer straight to the witness. An anchor-generated volume key, a per-box Tang instance plus the anchor-held second factor, unlocked by bpb's own unlock client. Heartbeat and events to the witness, alerts to a self-hosted ntfy. `bpb burn` and `bpb verify-burn` against a mock provider API. A minimal ledger. Keys are delivered over the pinned installer SSH session for now (trust on first use, recorded as such). Test:
   - the work volume unlocks with the anchor up and not with it down, and has exactly one keyslot plus its token
   - box A's WireGuard address cannot reach box B's Tang instance
   - the box's heartbeat still reaches the witness with the anchor and relay both stopped
   - killing the shipper raises an alert that reaches ntfy, and the witness's scheduled self-test delivers end to end
   - after `bpb burn`, the run's keystore entry is gone and an image of the box's disk taken before burn cannot be opened
   - burn refuses to proceed without pulled outputs unless `--abandon-outputs` is given, and the ledger records it
   - against a mock provider that leaves one snapshot and one auto-backup behind, `verify-burn` refuses to close the entry, and re-running `bpb burn` deletes them and converges
   - after a reboot mid-run, the work volume stays locked (the runtime keys lived only in RAM)
2. **M1. A real job on a real provider.** The same slice on Hetzner Cloud (T3), running a real job, with checkpoints pulled through the airlock and freeze/unfreeze/resume. Test:
   - **[hw]** an end-to-end job completes and its outputs pass the airlock
   - `bpb unfreeze` returns a frozen, still-running box to `running`; after a power-off, the run resumes on a fresh box from the last checkpoint with fresh keys
   - **[hw]** with nftables flushed as root on the box, the provider firewall still blocks everything except the relay and witness flows
   - **[hw]** rebooting the box from the provider's web console mid-run leaves it locked, the witness alerts and the run freezes, and the provider's console-session notification arrives where the operator reads it
   - **[hw]** GPU buffers the job allocated are zeroed before exit (on a GPU instance)
3. **M2. Bootstrap and attestation.** The installer host key pinned out of band, the single-use enrollment credential, nonce-bound attestation, sealed key delivery, host certificates issued after attestation, witness registration of the box's event key, and the witness "stream healthy" receipt gating Tang. Test:
   - an attestation response carrying an old nonce is rejected
   - the enrollment credential works once and is refused on any second use
   - after delivery the anchor holds no private half of any box key
   - the anchor refuses to enable a box's Tang instance without a fresh witness receipt
   - a box presenting a host certificate for another run ID, or none, is refused
   - a box that reboots cannot rejoin or unlock
   - **[hw]** a TPM quote including PCR 11 and PCR 12 verifies to the manufacturer's endorsement chain; a quote from a virtual TPM is rejected; a non-empty command-line override (PCR 12) fails attestation
4. **M3. Network and SSH hardening.** Rosenpass with an initial PSK, the box's network sysctls, the SSH profile, the listener inventory, the anchor's per-peer firewall and DNS resolver, the witness's TLS, and deploy-rs rollback for the long-lived machines. Test:
   - `curl 1.1.1.1` fails and appears in the anchor egress log
   - `wg show` reports a non-zero PSK, and nothing but the Rosenpass exchange passes before the PSK is set
   - serial-console login fails
   - the box's job-traffic endpoint and the provisioning source address are the relay's, never the anchor's
   - `ss -tulpn` lists exactly WireGuard, Rosenpass and sshd on `wg0`; a test service started on `0.0.0.0` raises a listener event even though nftables still blocks it
   - SSH: a client offering only a non-allowlisted KEX, cipher or MAC is refused; `ssh -G` and `sshd -T` dumps match the SSH profile table, including `compression no` and `disableforwarding yes`; an ssh-audit scan finds nothing outside the profile; a bare public key is refused and only a certificate works; the anchor's client refuses a box with no host certificate, or an expired one
   - the anchor's per-peer rules hold: the relay reaches nothing on the anchor, and a box reaches only its own Tang instance, the collector and the resolver; a box's WireGuard peer roaming to a new address raises an alert
   - the box's network sysctls match the profile, and a spoofed-source packet is dropped and logged as a martian event
   - the anchor's resolver refuses a DNSSEC-bogus answer and sends nothing upstream on port 53; NXDOMAIN on a non-allowlisted name is an event
   - the witness's ntfy endpoint refuses TLS 1.2 and logs the group each client negotiates
   - a relay configuration that stops forwarding fails its health check and rolls back within the confirmation timeout, while running boxes keep streaming evidence to the witness throughout
   - **[net]** CI records the group negotiated with each TLS upstream and flags any that fall back from `X25519MLKEM768`
5. **M4. Sensors, witness independence and alarm correlation.** auditd, LSM-hook sensors, both evidence paths, gap resend, nonce-bound heartbeats, hardened telemetry ingest, failed-authentication events, witness verdicts, the anchor–witness mutual watch, correlation, witness-enforced retention, and CA-signature logging. Test:
   - killing the eBPF shipper still leaves journald forwarding, and vice versa
   - a honeyfile read through io_uring (in a job profile that keeps io_uring) still fires
   - a dropped event that the resend fills raises no tripwire; a gap that cannot be filled freezes the run
   - a replayed heartbeat (stale nonce) is rejected
   - a fuzzed and oversized event stream is stored raw, rejected by the parser, raises an alert, and leaves the witness's history intact
   - each rejected SSH attempt (expired certificate, foreign CA, unknown principal, honey user) arrives at the witness with its reason
   - `nft flush ruleset` as root changes the ruleset digest and freezes the run; nothing burns automatically
   - stopping the anchor and relay together yields one "network event" alert and no per-box freezes, and boxes keep streaming evidence
   - a witness that stops alerting is caught by the anchor's watch of it
   - a host-certificate signature whose principal matches no started run raises an alert
   - the anchor's attempt to prune the witness's copy early is refused
   - **[hw]** GPU load with no job running raises a resource alert, and nothing on the automatic ladder fires
6. **M5. Lockdown and kernel config.** The arm-time switches, the box's tailored kernel, every module (NVIDIA and `nvidia_uvm` included) loaded before arm, BPF closed to root after arm, the LSM list, Landlock in bpb's services, and the job unit's sandbox. Test:
   - `/sys/kernel/security/lsm` lists `landlock`, `lockdown`, `yama` and `bpf`
   - after arm: `insmod` of any module fails; a root `bpf()` program load from anything but the shipper's identity is denied; `kernel.unprivileged_bpf_disabled` reads `1` and cannot be changed; `io_uring_setup` returns `ENOSYS` (compiled out)
   - the arm step succeeds on a kernel whose `kexec_load_disabled` sysctl does not exist
   - **[hw]** the NVIDIA modules, including `nvidia_uvm` and its device nodes, are loaded and working after arm, with no setuid `nvidia-modprobe` wrapper present; lockdown can be raised at runtime after they have loaded
   - **every sensor still fires after lockdown**, which decides `confidentiality` versus `integrity`
   - `kernel-hardening-checker` passes on config, cmdline and sysctls with only allowlisted exceptions
   - each bpb service fails to read outside its Landlock set
   - executing a copied binary from the work volume or `/tmp` fails; an interpreter running a script from outside the store raises an event; `vm.memfd_noexec` reads the value the job profile records
   - `kernel.core_pattern` is a pipe to a discarding program, `fs.suid_dumpable` is 0, and a crashing process leaves no core file anywhere
   - the arm snapshot covers `/etc/ld-nix.so.preload`
   - the job unit cannot write outside its work directory, gain a capability, or exceed its memory and task limits; its JIT cache (if the profile has one) is the only exec mount and every object loaded from it is an event
   - panic on oops is set on the anchor, witness and relay and not on the GPU box
7. **M6. Verity image and closure minimization.** The deployment path (`nixos-anywhere --phases kexec`, then the image written to disk), the read-only `/etc` overlay, the deny list, and the closure gates. Test:
   - the box boots from a verity image written by the kexec installer; flipping one byte in the store partition makes the read fail
   - `/etc` is a read-only overlay and a write to it fails; `/etc/machine-id` differs across two boots
   - every writable mount except `/run/wrappers` (and the job's JIT cache, if any) is `noexec,nosuid,nodev`; `/run/wrappers` matches its allowlist and no other file carries setuid, setgid or file capabilities
   - root is tmpfs; no swap; all four sleep targets masked; the unused systemd components are absent; there is no sudo, no Nix and no default packages
   - each deny-list pattern matches the package it targets and nothing in a known-good closure; a closure containing a denied package fails the build
   - the closure contains no interpreter except bash and the job's pinned one (and only bash when the job runs in a microVM)
   - a dependency change that adds a new store path fails CI until it is reviewed; CI fails if any bpb unit's `systemd-analyze security` score is above the agreed threshold
   - attestation rejects the debug flavor's closure hash
   - `vulnix` and Lynis produce reports for review; CI fails only on a confirmed, reachable critical
8. **M7. Canaries.** Randomized layout, honeyfiles, per-box honey credentials minted from a dedicated account of your own, DNS canaries, the honey user and the honey port. Test:
   - each `redteam` case for canaries fires (honeyfile open, honey login, DNS canary resolution, honey-port touch logged by nftables)
   - no honey AWS key decodes to a known public canary account ID
   - the ledger's `honey_cred_ids` records each box's honey credentials, and a simulated external use attributes to the right box
9. **M8. Ledger and airlock.** Salted commitments, witness-enforced retention, the airlock's outbound and return paths, known-answer items, and anchor-side send hashes. Test:
   - pruning deletes raw fields and salts from the anchor copy, the witness copy and the newest backup, the commitments remain, and the chain and signatures still verify
   - outbound: metadata is stripped (EXIF, PNG chunks, timestamps, xattrs), names are HMAC tokens, item count is padded to the bucket, and order is shuffled
   - return: a result with extra keys is rejected; path- and URL-shaped text is stripped; nothing leaves quarantine before validation passes
   - a run whose known-answer items come back wrong is rejected and frozen
   - `run_key` and the token mapping are gone after burn
   - each run's bundle carries its own unique content canary, recorded in `canary_ids`, and a simulated sighting attributes to the right run
10. **M9. Anchor hardening and recovery.** Two tokens, one key per role, the per-run keystore excluded from backups, `age` backups, the airlock and collector microVMs, the VM-host side-channel settings, and the independent rebuild. Test:
    - the seizure drill passes (see Anchor)
    - the restore drill passes using only the spare token, on blank hardware, from the newest backup
    - no backup contains any per-run keystore entry
    - the airlock and collector microVMs cannot read anything outside their own store and the one key each is given
    - the independent rebuild flags a box closure whose `narHash` differs from the anchor's build
    - the anchor reports SMT off, L1 flush on VM entry, and page table isolation on
11. **M10. Confidential tier.** SEV-SNP or TDX with direct measured launch of the bpb UKI (no kexec), `REPORT_DATA` binding, certificate-chain, TCB and policy checks, plus H100 confidential-mode GPU attestation for GPU jobs. Test:
    - **[hw]** the launch measurement matches the bpb UKI built on the anchor; a report with the wrong `REPORT_DATA` is rejected; a box that reached its kernel by kexec fails T4 attestation
12. **M11. OS diversity.** Witness and relay move to OpenBSD, with the ingest parser under pledge/unveil. Test: the M3 relay tests and the M4 witness tests pass against OpenBSD VMs booted from a custom QEMU harness.

The red-team lab is not a single milestone. Its scenario 1 (honest baseline) and scenario 5a (rigged RNG) belong in M0; the attestation-lie scenario in M2; the RAM-dump, snapshot and console scenarios alongside M1 and M6; the network scenario in M4. Each lands with the control it tests, so the lab grows with the plan rather than as a wall at the end.

## Open questions

- Witness hosting. A Pi at home shares fate with the anchor's network, and anyone who seizes the anchor takes it in the same trip, so it cannot be the only witness of the anchor. A tiny VPS is a third provider to trust. Maybe both, with alerts from either.
- The verity deployment path (`nixos-anywhere --phases kexec`, then writing the image to disk) is the plan, but untried. M6 proves or replaces it.
- Whether the relay and the witness may share a VPS. Fewer providers to trust, but one compromise then costs both the anchor's cover and its watcher.
- Firmware for the anchor (and a home witness) on T1 hardware. Rung 9 includes an anchor that is seized, backdoored in firmware and handed back. ownerboot (from the sixos author) builds coreboot, a kernel and a cryptsetup initramfs entirely from source with Nix, with binary caches disabled (`--option trusted-public-keys ""`) and binary-only packages forbidden.[^ownerboot] Everything lives in the flash chip, so with the write-protect pin shorted, booting touches no writable unencrypted media. Normal and fallback images can each be write-protected independently, and it kexecs a signed second-stage kernel. It would make the anchor's firmware reproducible and physically read-only. Two things stop it being a decision today:
  - supported hardware is old and narrow: ASUS KGPE-D16 (Opteron-era server board), MSI AM1I, and the Samsung Chromebook Plus (arm64)
  - by the author's own account it currently "dumps you at a bash prompt": the init that hands off to the real system is unpublished

  Heads is the better-known alternative for mainstream hardware and has not been researched yet.
- Tetragon vs. hand-rolled bpftrace. Tetragon is heavier but enforces, not just observes.
- Whether the box should ever run a GPU stack in a container, or whether the closure should own the driver too. Owning it is more verifiable and more painful. The container route adds the NVIDIA Container Toolkit's privileged hooks to the attack surface (NVIDIAScape, CVE-2025-23266).
- Whether T3c (container-only marketplaces) is worth supporting at all, or whether bpb should refuse anything it cannot kexec into.
- Upstream NixOS boot security (nixpkgs #549372): autopen, deterministic module signatures (#571181) and stage 2 measurement. When they land, module signing joins ordering as a control on the box, and a measured mutable store may suit the anchor. Check the issue before M5 and again before M10.
- The NixOS STIG (Anduril, on stigviewer.com) has not been reviewed yet. Diff it against this plan before M5.
- Register-based disk-key storage to shrink the cold-boot window on bare metal. TRESOR keeps the AES key in CPU debug registers so it never reaches RAM;[^tresor] cold-boot remanence persists seconds to minutes on DDR4/DDR5 even without cooling.[^coldboot] It only helps on T1/T2 against physical RAM capture (a hypervisor reads registers too, so it does nothing on T3), and a current maintained implementation on a recent kernel has not been found. Worth a look for the bare-metal tiers; not a control until something maintained exists.
- How much cover traffic is worth it. Probably fixed-size job buckets and no more.
- Whether KVM-VMI's patched kernel and QEMU are worth standing up for the red-team lab, or whether `pmemsave` plus the gdbstub already answer "can the host read a key from RAM." LibVMI matters only for structured, continuous introspection; the simple tools suffice for the leak-or-not oracles.
- Whether the lab should emulate BMC/firmware persistence (the Cloudborne case) at all. It is hard to emulate faithfully, so it may stay a documented, untested threat rather than a lab scenario.
- dm-integrity on the LUKS work volume. LUKS alone gives confidentiality, not integrity, so a host can flip ciphertext bits (lab scenario 6). The work volume is single-use and never reused across runs, which limits the value, but dm-integrity would turn tampering into a detected fault. Decide after scenario 6 shows whether undetected tampering buys the host anything.
- Decided: automatic burn is not safe. It destroys undrained results and the evidence of an intrusion, and most of its triggers are self-reported by the box. Automatic actions stop at freeze; burn is a human command (see Response ladder).

---

[^ownerboot]: amjoseph, ownerboot README (last commit checked: June 26, 2026). https://codeberg.org/amjoseph/ownerboot

[^tresor]: Müller, Freiling and Dewald, "TRESOR Runs Encryption Securely Outside RAM" (USENIX Security 2011), keeping the AES key in CPU registers rather than DRAM. https://www.usenix.org/legacy/events/sec11/tech/full_papers/Muller.pdf

[^coldboot]: Halderman et al., "Lest We Remember: Cold-Boot Attacks on Encryption Keys" (USENIX Security 2008), and Kicksecure's Cold Boot Attack Defense page summarizing 3MDEB's 2024–25 DDR4/DDR5 retention findings. https://static.usenix.org/event/sec08/tech/full_papers/halderman/halderman_html/ and https://www.kicksecure.com/wiki/Cold_Boot_Attack_Defense
