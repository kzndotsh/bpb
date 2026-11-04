# Tiers

## Escalation tiers (pick per job)

| Tier | Where | What it buys |
|---|---|---|
| T0 | QEMU on your own machine | Learning, CI, all tests |
| T1 | Hardware you own, own VLAN | Nothing leaves your custody |
| T2 | EU dedicated metal (Hetzner/OVH) | No hypervisor, GDPR, TPM sometimes |
| T3 | GPU marketplace (Vast/RunPod datacenter tier) | Cheap burst, operator root assumed |
| T3c | GPU marketplace, container only | Cheapest burst, most controls gone |
| T4 | Confidential VM (SEV-SNP/TDX), + NVIDIA GPU CC where the job needs a GPU | Host software cannot read RAM, launch attested |

The tier tells you how much the host can see. A second axis, the compute profile, tells you where T4 even exists, because confidential computing for accelerators is far behind the CPU. Pick both.

The same flake targets T0 through T4, but the tiers differ in more than `attest.nix` and the provider burn hook:
- T2 can run the job in a microVM, which changes the deny list (no interpreter but bash on the box) and needs the VM-host side-channel settings (SMT off, L1 flush, page table isolation).
- Secure Boot, a physical TPM and a provider firewall exist on some tiers and not others, and each changes what attestation and the console-reboot defence can claim.
- T4 cannot use kexec at all (below).

T3c cannot run the flake at all.

Tier caveats:

- **T2.** Bare metal removes the hypervisor but not the BMC. In the 2019 Cloudborne research, a BMC firmware change on IBM SoftLayer survived reclamation and reached the next tenant, along with the BMC root password and logs.[^cloudborne] Assume the previous tenant may own the BMC. LUKS + Tang still limits it to while the box is running.
  - Bare metal also allows something cloud VMs usually do not: the job runs inside a microVM (microvm.nix) on the box[^microvm], with its own read-only store holding only the job's closure. The box's sensors then sit below the job's privilege, so a compromised job is rung 1 from the sensors' point of view, not rung 2 (principle 6). Most cloud VMs do not offer nested virtualization, so on T3 the job runs directly on the box. When the box hosts the job's microVM, it takes the same VM-host side-channel settings as the anchor (SMT off, L1 flush, page table isolation). SMT off roughly halves the available CPU threads, so this is a per-job decision that the ledger records.
- **T3.** Vast.ai's default instances and RunPod pods are Docker containers that share the host kernel: no kexec, no `nixos-anywhere`, no lockdown, no own sensors below the container. Vast's KVM VMs give a real kernel but launch only from fixed Ubuntu images[^vast], so check per provider that kexec from inside works before counting it as T3. Anything that only offers containers is T3c.
- **T3c.** Only the airlock, tokenization, decoy padding, canaries in the data bundle and egress through the anchor (if the provider allows a WG client in the container) still apply. Treat it as "send only what you would publish". If you run your own GPU container stack anywhere, keep the NVIDIA Container Toolkit at 1.17.8+: NVIDIAScape (CVE-2025-23266) let a crafted image escape to root on the node through the toolkit's hooks.[^nvidiascape]
- **T4.** It protects RAM from host *software*, not from someone with the hardware, and not forever. Three limits, each real:
  - **Its own attestation has been broken, and will be again.** RMPocalypse (CVE-2025-0033, CCS 2025) let a malicious host forge SEV-SNP attestation reports until a 2025 microcode/firmware fix.[^rmpocalypse] The anchor's minimum-TCB check (see the `attested` state) rejects hosts below the fix, but that only works until the next break, and the box cannot verify the host's patch level from inside. Treat T4 as "the host must beat a published bar and keep beating it", not "the host cannot read RAM".
  - **Physical access still wins, now on DDR5.** Battering RAM (a sub-$50 interposer) and WireTap broke SGX and reportedly SEV-SNP on DDR4; TEE.Fail (CCS 2025) extended the interposer line to DDR5 and extracted the ECDSA key behind Intel's TDX attestation chain.[^dram][^teefail] Intel and AMD treat physical attacks as out of scope. A provider with the hardware is not stopped by T4.
  - **T4 cannot use the kexec install path.** A cloud confidential VM's launch measurement covers the image the provider launched, typically its firmware or paravisor; kexec into the bpb kernel replaces what was measured, so the measurement says nothing about the bpb UKI.[^kexecsnp] T4 needs measured direct boot (the VMM hashes the UKI into the firmware's measured region, which stock cloud OVMF may not support) or the provider's own vTPM-based attestation of what booted.

  So T4 raises the bar against a snooping host; it is not a wall, and the honest ceiling says so.

## Compute profiles (what the job runs on)

Confidential computing is mature for the CPU and immature-to-absent for accelerators, so the accelerator the job needs decides whether a RAM-protected tier exists at all.

- **No accelerator (CPU-only job).** The clean case. T4 is a plain SEV-SNP or TDX confidential VM, no GPU attestation in the loop, and it is the most widely offered (Azure, GCP, AWS, and self-launchable on your own EPYC/Xeon for T1). If a job can be CPU-only, this is the profile with a real end-to-end confidential path.
- **NVIDIA GPU (CUDA, the ML default).** T3 is the cheap marketplace (Vast, RunPod, Lambda) with no RAM protection, operator root assumed. T4 exists but only on the big clouds: H100/H200 confidential mode is GA on Azure (NCCads H100 v5) and GCP (a3-highgpu-1g), each pairing a CPU TEE (SEV-SNP on Azure, TDX on GCP) with the GPU's own TEE and encrypted VRAM.[^nvidiagpucc] Attestation is composite: CPU evidence to the cloud's or your verifier, GPU evidence to NVIDIA's NRAS or a local verifier, and the anchor must check both.[^nras] The catch for bpb's cost model: "move this GPU job from T3 to T4" means leaving the cheap marketplaces for an expensive, KYC'd big-cloud instance. For a GPU job, T4 is a different provider class, not a config flag.
- **AMD GPU (MI300X, ROCm).** Confidential mode exists but, per secondary sources, sits inside the host VM's SEV-SNP boundary rather than being a separate per-GPU TEE with its own attestation of firmware and VRAM state. No clearly rentable confidential MI300X path was found, and ROCm's ML ecosystem is thinner than CUDA. Watch, do not plan on it; recheck against AMD's own Instinct documentation before counting it.
- **Intel GPU (Max/Flex/Arc/iGPU).** No confidential-computing path for Intel's own GPUs. Intel's confidential-GPU story is TDX on the CPU paired with an *NVIDIA* GPU through Intel Trust Authority, not Intel silicon doing the GPU TEE.[^intelgpucc] So "Intel compute" means CPU-only TDX with no accelerator confidentiality: if the job needs an Intel GPU or iGPU, there is no T4 for it, only T3-style trust-the-host.

The decision rule: a CPU-only job can have real confidential compute anywhere; an NVIDIA job can, but only on a big cloud at big-cloud prices; an AMD or Intel GPU job currently cannot, so it runs at T3 with the data-side controls (airlock, tokenization, decoy padding) as its only protection, exactly as the honest ceiling describes.

---

[^cloudborne]: BleepingComputer, "Hackers Backdoor Cloud Servers to Attack Future Customers" (February 2019), on Eclypsium's Cloudborne research. https://www.bleepingcomputer.com/news/security/hackers-backdoor-cloud-servers-to-attack-future-customers/

[^microvm]: (primary) astro, microvm.nix. https://github.com/astro/microvm.nix

[^vast]: Vast.ai docs, Linux Virtual Machines. https://docs.vast.ai/linux-virtual-machines

[^nvidiascape]: Upwind, "Understanding the NVIDIAScape CVE-2025-23266 Container Toolkit Vulnerability" (2025). https://www.upwind.io/feed/understanding-the-nvidiascape-cve%E2%80%912025%E2%80%9123266-container-toolkit-vulnerability-and-why-your-ai-workloads-are-most-likely-safe

[^rmpocalypse]: RMPocalypse (CCS 2025), demonstrating SEV-SNP attestation forgery, code injection and register-state replay. https://rmpocalypse.github.io/rmpocalypse-CCS2025.pdf

[^dram]: Keysight, "Security Highlight: DRAM Interposer Attacks on Confidential Computing" (October 2025), on Battering RAM and WireTap. https://www.keysight.com/blogs/en/tech/nwvs/2025/10/22/security-highlight-dram-interposer-attacks-on-confidential-computing

[^teefail]: "TEE.Fail" (ACM CCS 2025): a ~$1,000 DDR5 memory-bus interposer that extracted the ECDSA key behind Intel's TDX attestation chain, extending the Battering RAM / WireTap interposer line to DDR5. https://tee.fail/

[^kexecsnp]: kexec mailing list, "x86/snp: Add kexec support" (May 2024): the launch measurement indicates the directly booted environment, and a kexec'd kernel runs an environment the launch never measured. https://lists.infradead.org/pipermail/kexec/2024-May/029990.html

[^nvidiagpucc]: Zhu et al., "Blueprint, Bootstrap, and Bridge: A Security Look at NVIDIA GPU Confidential Computing" (arXiv 2507.02770), on the H100 GPU TEE paired with a CPU TEE and its attestation. https://arxiv.org/pdf/2507.02770

[^nras]: NVIDIA Remote Attestation Service (NRAS) release notes (GPU attestation; GA since 2023). https://docs.nvidia.com/attestation/cloud-services/latest/nras/nras_releases.html

[^intelgpucc]: Intel Trust Authority, "GPU Remote Attestation" (Intel's confidential-GPU path is TDX on the CPU plus an NVIDIA GPU TEE; a GPU alone is not a complete TEE). https://docs.trustauthority.intel.com/main/articles/articles/ita/concept-gpu-attestation.html
