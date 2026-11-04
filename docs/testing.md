## Red team mode (the lever)

`bpb redteam <box>` runs a scripted adversary against your own box and asserts each sensor fired at the witness:

- open a honeyfile
- add a line to `authorized_keys`
- load a bpf program, try `insmod`
- resolve a DNS canary
- `curl` an address outside the allowlist
- kill the log shipper
- write to `/etc/ld-nix.so.preload`
- `nft flush ruleset`, then `iptables -I INPUT -j ACCEPT`
- run a CPU or GPU burn while the job is idle
- read a honeyfile and open a socket through io_uring (must fail with io_uring off, and must still fire where it is allowed)
- log in on the serial console
- SSH in with an expired certificate, a certificate from another CA, an unknown principal, and as the honey user
- after burn, run a Tang recovery against the burned box's binding (must fail)

Each case is also a `nixosTest` VM test in `tests/`, so the whole suite runs in CI with no cloud account. A control without a firing test does not count as a control. This is where most of the learning happens.

`redteam` plays an adversary *on* the box (rungs 1–3). The lab below plays the adversary *under* it (rungs 4–6), which is the only position a rented provider actually occupies and the one the honest ceiling leans on hardest.

## Red-team lab: the hostile host

The threat model concedes the host has root from second zero, yet nothing so far tests what that costs, because on a real provider you cannot see from the host side. The lab makes the host side yours. It runs entirely on hardware you own, on your own VMs; it never touches a real provider's infrastructure. Its job is to turn the plan's rung-4-to-6 claims into pass/fail facts, including the claims that are admissions of defeat.

**You are the hypervisor.** Your machine is **L0**, running KVM/QEMU and playing the provider. A bpb box boots as an **L1 guest**, with the anchor, relay and witness as their own VMs so the whole topology exists. Nested virtualization (`kvm_intel nested=1`) carries the caveat that it is officially unsupported and can be flaky,[^nestedkvm] so pin the kernel and QEMU versions and expect the odd rerun. The point of being L0 is that every "the host can…" in the threat model becomes something you actually do and then measure.

**The provider's powers, as lab tools.** Each is a real provider capability mapped to a tool, so the lab exercises the actual threat, not a caricature of it:

| Provider power | Lab tool |
|---|---|
| Read guest RAM while it runs | QEMU monitor `dump-guest-memory` / `pmemsave`, `virsh dump`, or LibVMI via KVM-VMI for structured live reads[^libvmi][^kvmvmi] |
| Read registers, single-step | QEMU gdbstub (`-s -S`), physical-memory mode[^qemugdb] |
| Image the disk at any time | copy or snapshot the qcow2 while running |
| Keep a snapshot after "deletion" | retain a qcow2 snapshot past burn |
| Console / rescue boot | virsh console / VNC, firmware control |
| Lie in attestation | a controlled swtpm / fake vTPM that signs anything; the SNP-forgery literature is the reference for the hardware-quote case[^snpguard][^rmpocalypse] |
| Pause the VM | `virsh suspend`, for cold-boot-style capture |
| Rig randomness | a biased virtio-rng backend |
| Be the network | `tc`/Pumba on the L0 bridge to drop, delay, reorder |

**Scenarios, one per "what if", each with a hard oracle:**

1. **Honest provider (baseline/control).** A normal run. Assert: the witness sees a clean stream, burn completes, a post-burn disk image will not open. Everything else is measured against this.
2. **The provider lies about attestation.** L0's fake vTPM returns a valid-looking quote for a box image you secretly modified (the Hansa case). Assert: the anchor rejects it, because the endorsement chain is not a real manufacturer's, or the nonce / PCR-12 binding is wrong.
3. **The provider has more access than it admitted.** L0 dumps guest RAM before arm, mid-run and just after burn. Assert: the post-burn dump holds no usable volume key; the mid-run dump *does* (which the plan concedes, so this confirms the ceiling, not a defence); the box never noticed (also conceded).
4. **The provider kept a snapshot.** L0 retains a snapshot past burn. Assert: `verify-burn` passes only when the mock provider API reports it gone, and a burn that deletes only the instance fails.
5. **A bug or vuln on the provider side.** (a) Rigged RNG: assert the volume key is still unpredictable, because it is generated on the anchor. (b) Console escape: assert `init=/bin/sh` and a modified `usrhash=` succeed without Secure Boot (the known limitation) and fail with it or with a PCR-12 quote, and that either way the box cannot rejoin or unlock and the witness alerts.
6. **The provider tampers with the disk.** L0 flips bytes in the running qcow2. Assert: verity faults on a store read; the integrity-free LUKS work volume is where this scenario decides whether the dm-integrity open question needs closing.
7. **The provider is the network.** L0 drops, delays and reorders the box's packets. Assert: gap-resend tolerates loss, a sustained drop freezes rather than burns, and cutting the anchor path does not silence the box's evidence to the witness.

**Framework: two layers, one manifest.** These are two different kinds of work and they want two tools, glued by a shared scenario manifest (plain YAML: scenario, fixture, action, oracle) that is the single source of truth, so the two layers cannot drift apart.

- **Deterministic layer.** The scenarios above have hard oracles, so they are not an AI-eval problem; AI-eval frameworks score fuzzy output. They run on the `nixosTest` driver that already drives bpb's multi-VM tests, with Avocado-VT[^avocadovt] underneath where a scenario needs QEMU-specific manipulation, and Stratus Red Team[^stratus] for the provider-API scenarios (snapshot kept, instance not deleted), since it is the cloud-focused adversary-emulation tool. Fixtures are golden box images snapshotted at each lifecycle state.
- **Agentic layer.** Open-ended probing ("you are a hostile host; find anything the box leaks that the witness does not catch") is where an AI-eval framework fits. Inspect AI[^inspect] is built for it: its dataset→solver→scorer model maps onto scenario→agent→oracle, it ships a sandbox and `react()` agents with shell tools, and it logs transcripts to review for leads. Harbor[^harbor] (from the Terminal-Bench team) is a close alternative, a clean `dataset/model/agent` runner that scales to many parallel environments; its edge is throughput, Inspect's is the explicit scorer and transcript viewer, and either works since both default to Docker and must reach out to drive the libvirt VMs regardless. Cybench[^cybench] is the reference for structuring such tasks (subtasks for partial credit, several agent scaffolds), to borrow rather than adopt. The agent only ever drives L0 and lab VMs, never a real provider.
- **The discipline, unchanged from `redteam`:** an agent finding something is a lead, not a result. Every lead becomes a deterministic scenario with a hard oracle before it counts. The report is a scenario × claim matrix, and its "known-limitation" cells (RAM readable mid-run, console escape without Secure Boot) are as important as its passes: they are the honest ceiling made concrete.

---

[^nestedkvm]: Nested KVM: enabling guide and Red Hat's note that nested KVM is unsupported and not for production. https://computingforgeeks.com/how-to-enable-nested-virtualization-on-kvm-qemu/ and https://www.redhat.com/ja/blog/inception-how-usable-are-nested-kvm-guests

[^libvmi]: LibVMI, a virtual machine introspection library for reading a running guest's memory. https://github.com/libvmi/libvmi

[^kvmvmi]: KVM-VMI (patched kernel, patched QEMU, Nitro, LibVMI) for introspection on KVM. https://github.com/KVM-VMI/kvm-vmi

[^qemugdb]: QEMU gdbstub documentation (`-s`/`-S`, register and memory inspection, physical-memory mode). https://qemu.googlesource.com/qemu/+/9ccb69df554a5204077cda101b7bcf0f19544553/docs/system/gdb.rst

[^snpguard]: SNPGuard, open-source remote attestation of SEV-SNP VMs (nonce in the report's guest-data field for replay protection). https://themoonlight.io/review/snpguard-remote-attestation-of-sev-snp-vms-using-open-source-tools

[^rmpocalypse]: RMPocalypse (CCS 2025), demonstrating SEV-SNP attestation forgery, code injection and register-state replay. https://rmpocalypse.github.io/rmpocalypse-CCS2025.pdf

[^avocadovt]: Avocado-VT, a QEMU/KVM and libvirt test framework. https://github.com/avocado-framework/avocado-vt

[^stratus]: Stratus Red Team (Datadog), cloud-focused adversary emulation ("Atomic Red Team for the cloud"). https://github.com/DataDog/stratus-red-team

[^inspect]: Inspect AI (UK AI Security Institute), an eval framework with a docker sandbox and `react()` agents; its tutorial includes a CTF example. https://inspect.aisi.org.uk/tutorial.html

[^harbor]: Harbor (laude-institute), an agent-evaluation framework from the Terminal-Bench team: runs agents in sandboxed environments, defaults to Docker, scales across cloud sandbox providers. https://www.harborframework.com/ and https://github.com/laude-institute/harbor

[^cybench]: Cybench (Stanford, ICLR 2025), a framework for agentic cybersecurity tasks with subtask scoring and multiple agent scaffolds. https://arxiv.org/html/2408.08926v2
