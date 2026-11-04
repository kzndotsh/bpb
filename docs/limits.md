# Limits

## Forensic exposure

What an examiner recovers from a seized or imaged bpb machine, and the control that answers it. This is the adversary half of the threat model written down: published DFIR and sanitization research, not a method. The reassuring result is that nothing below needs a control the plan does not already have; the research mostly confirms the ceiling stated in the next section.

| State | What is recoverable | What answers it |
|---|---|---|
| Powered-off box, Tang key deleted | The LUKS2 header still reveals format version, cipher and mode, keyslot count and a UUID, so an examiner sees that the volume is encrypted and how, but not the key or plaintext.[^luksdump] There is no passphrase to brute-force (the keyslot secret is derived from the Tang exchange plus the anchor-held secret) and, after burn, neither the Tang key nor the anchor-held secret exists anywhere, including in backups. A future quantum attacker could re-derive the Tang half from the header's public values, but not the anchor-held half. If burn step 3 completed, there is no readable header at all. | Network-bound unlock plus key deletion at burn, with the post-quantum second factor. This is cryptographic erase, the one sanitization method that works on flash.[^nist80088] |
| Powered-off box, Tang key still live (a box seized mid-run, before burn) | Same header exposure, but the volume is openable by whoever can complete an unlock: reach this box's Tang instance and obtain the anchor-held secret. A seized, powered-off disk cannot do either: the runtime WireGuard key that could reach them lived only in RAM. Someone who imaged the box's RAM while it ran holds that key, which is why each box's Tang instance accepts only that box's address, roaming raises an alert, and a second recovery for a run is a tripwire. | Per-box Tang instance firewalled to its own box; RAM-only runtime keys; one recovery per run. |
| SSD-level recovery | Built-in secure erase is unreliable on flash. In the UCSD FAST '11 study, the drive's own erase cleaned only 4 of 12 SSDs, one reported success while data survived, and single-file overwrite failed outright, because the flash translation layer, wear-leveling and over-provisioning keep copies in cells the OS cannot address.[^ssderase] | Why burn deletes the Tang key first and treats `luksErase` and `blkdiscard` as defense in depth, never the guarantee. Crypto-erase does not depend on reaching every cell. |
| Running box, RAM | The real exposure. Cold-boot remanence persists seconds on DDR4 and DDR5 without cooling, and up to ~2 minutes on at least one DDR4 module (3MDEB, 2024–25); DDR4 memory scramblers were shown insufficient in 2017.[^coldboot][^scrambler2017] A hypervisor needs no cold boot at all: it can pause a VM and image its RAM and CPU state directly (LibVMI on KVM/Xen).[^vmiforensics] | Rung 4+. Keep less in RAM and for less time, suspend disabled so keys are not held across sleep. Only T4 confidential compute changes the hypervisor case, and nothing inside the box changes the physical-RAM case. The honest ceiling. |
| Timeline and log artifacts | Even with timestamps wiped, examiners rebuild activity from filesystem journals (ext4 jbd2, XFS) and journald; the 2025 FJTA tool reads file activity straight from the journal, independent of MACB timestamps.[^fjta] Bash history can also linger in the in-memory buffer until flushed. | tmpfs root (no persistent filesystem journal to carve), `/persist` empty by default, journald volatile and forwarded off-box, `HISTFILE=/dev/null`, no swap. |
| Provider-kept disk snapshots | Snapshots and auto-backups outlive the volume and are routine to acquire.[^snapshot] | The snapshot/image/backup deletion step in burn, and `verify-burn` gating the close on an empty listing. |

The one genuinely new idea the research surfaces is register-based disk-key storage (TRESOR-style), which keeps the volume key in CPU registers instead of RAM to shrink the cold-boot window on bare metal. It is an open question below, not a control, because a current maintained implementation has not been confirmed.

## The honest ceiling

Write this into the README so nobody, including future you, oversells it:

- The host sees whatever bytes you send it, and the job's size and timing.
- Hypervisor RAM reads are invisible from inside. Only T4 changes that, and only partly: T4's attestation has itself been forgeable (RMPocalypse, 2025), it protects nothing against a host with a memory-bus interposer (now demonstrated on DDR5 by TEE.Fail), and its guarantee lasts only while the host stays above the patch bar the anchor checks. T4 makes a snooping host work much harder; it does not make RAM unreadable.
- Confidential compute for accelerators lags the CPU. A CPU-only job can run confidentially almost anywhere; an NVIDIA GPU job can only on an expensive big-cloud instance; an AMD or Intel GPU job cannot yet at all, so it falls back to T3 with only the data-side controls.
- The host controls the box's clock and its randomness. Keys come from the anchor for that reason.
- BMC/IPMI on metal can read disk and screen, and may carry a previous tenant's implant. LUKS + Tang limits that to while the box is running.
- Without a hardware quote, attestation is the box's word. A careful root attacker can report the right hash.
- On container-only marketplaces most of this plan does not run. Only the data-side controls survive.
- Billing ties the box to a person. bpb does not hide who runs it (see Non-goals and "Operator anonymity, in theory").
- The anchor is the single place where everything comes together. Its hardening raises the cost of taking it; a running anchor taken by someone with physical access still gives up what is in RAM.
- A careful root attacker on a T2/T3 box may evade sensors. Off-box evidence and canaries raise the cost; they do not make it zero. Such an attacker can read the heartbeat key from RAM and keep sending healthy-looking heartbeats. Periodic hardware quotes do not stop that, since root inside a correctly booted box can request a genuine quote: they catch a reboot or a swap, and on metal IMA logs any new executable. An attacker who stays in memory is caught by none of them.
- Anyone who can flood the witness's address can open a cover window. While the witness is dark, evidence waits in the box's buffer instead of having left, and a careful root holding the event key can rewrite what has not been sent. The fleet pauses arms and new data rather than freezing every box, so a flood costs throughput, not results. More than one witness narrows the window (see Open questions).
- Without Secure Boot, the UKI's embedded command line is not authoritative: someone at the console can boot it with a different command line, including a different verity root hash. A TPM quote that includes PCR 12 detects that; on tiers with neither Secure Boot nor a TPM, nothing does. The box still cannot rejoin or unlock after that reboot.
- On tiers without hardware attestation, the first delivery of keys to a box is trust-on-first-use, and the host could intercept it. The ledger records which runs were delivered that way.
- The return path can check that results are well-formed and that known-answer items are right. It cannot prove that the rest of a result is true.
- Automatic responses stop at freeze, so an attack in progress is contained, not erased; burning it needs you. That is the price of not letting root on the box, or an outage, burn your results.
- NixOS has no mandatory access control today. The job runs confined by systemd sandboxing, noexec mounts and the verity store, not by SELinux or AppArmor. bpb's own services add Landlock; the job's own code does not.
- Nix's own tooling has no published security audit or risk assessment. bpb removes the Nix daemon from the box and keeps the anchor's Nix patched, but the build tooling itself is trusted, not verified.
- Module signing on NixOS is new: deterministic module verification merged in October 2026, optional and not yet a NixOS option. Until bpb adopts it, the boot window before the module lock relies on ordering alone.

The goal is a box where a successful snoop yields tokenized, decoy-padded, uncorrelatable bytes for a few hours, and where you probably find out anyway.

## Operator anonymity, in theory

Not implemented, and not planned (see Non-goals). This section exists to make the gap precise: what anonymity would demand, where bpb gives it up, and why bolting it on would not work.

### The goal, stated properly

Three properties, from weakest to strongest, against an adversary who can compel providers, ISPs, payment processors and phone vendors, and can watch a large share of the internet:

- **Anonymity.** No run can be tied to you.
- **Unlinkability.** No two runs can be tied to each other. Losing this is how anonymity usually dies: link enough runs together and one of them carries a mistake.
- **Unobservability.** Nobody can tell you use bpb at all.

bpb as designed offers none of the three. Its security model needs a fixed, trusted anchor, and anything fixed is a link.

### Why it is a different kind of problem

Security controls stack: each layer an attacker has to beat adds cost. Anonymity does not stack. It fails at its weakest link, and one identifier that connects two contexts undoes everything else. That is why "add more controls" cannot get there.

Three results from the literature set the shape:

- **The anonymity set.** You are hidden only among people who look like you. A bespoke, hardened, public flake is a fingerprint. If only a handful of people run it, any box that behaves like bpb points at a handful of people. Hardening and blending in pull in opposite directions.
- **Low-latency anonymity does not beat a global observer.** Tor-style networks defeat local observers. An adversary who can see both ends can match them by timing (Murdoch & Danezis 2005[^murdoch]; Johnson et al., "Users Get Routed", CCS 2013[^johnson]). Defeating that takes mixnets with cover traffic (Loopix, Nym), which cost latency and bandwidth.
- **Operators fail, not the cryptography.** Publicly documented cases end on one slip: Ross Ulbricht's early forum posts, one of which gave his personal Gmail address.[^ulbricht] Hector Monsegur is widely reported to have been found after a single IRC login without Tor, though he disputes that account and other routes have been reported.[^sabu] Over a long enough time, one slip is close to certain. A design that needs perfect operators for years has already failed.

### What documented failures show

Darknet-market takedowns from 2013 to 2025 are the largest public record of anonymity failing under a well-resourced adversary. It is tempting to read them as a list of careless people. The better reading is that each operator had to be careful for years, under constant attention, and needed only one slip. The risks recur, and they are the same links the table below names:

| Risk | What happened | Cases |
|---|---|---|
| A real-world identity, linked once | A personal email address on an early forum post, in a server lease, or in a site's own outgoing mail. One IP address shared between an anonymous account and personal accounts. | Silk Road,[^ulbricht] Silk Road 2.0,[^sr2] AlphaBay,[^alphabay] Kingdom[^kingdom] |
| The money trail | Funds ended at an exchange that knew the customer, or in wallets in the operator's own name, including after a swap to Monero. A tip jar paid out to the operator's own accounts. An exit scam created a fresh trail to follow. | Incognito,[^incognito] Monopoly,[^monopoly] Kingdom,[^kingdom] Dream Market (OxyMonster),[^oxymonster] Bohemia[^bohemia] |
| The access path | Admins reached the servers through commercial VPNs. The VPN provider's records put a suspect's home connection online at the same times as admin activity. | Wall Street Market[^wsm] |
| Writing style | Hundreds of forum posts matched an identifiable person's public posts by vocabulary, punctuation and language switching. | Dream Market (OxyMonster)[^oxymonster] |
| Devices at the wrong moment | A laptop seized unlocked and unencrypted, mid-way through an admin task on the server. Devices holding wallet keys seized at an airport. A laptop holding the account's PGP key searched at a border. | AlphaBay,[^alphabay] Bohemia,[^bohemia] Dream Market (OxyMonster)[^oxymonster] |
| People inside | An undercover agent on the support staff for months, paid by the operator. | Silk Road 2.0[^sr2] |

None of these cases needed a cryptographic break. Each risk is a link in the sense used below, and each is part of why operator anonymity stays a non-goal: bpb can avoid adding links, but most of these live in the operator's life, not in the infrastructure.

### Where bpb links you

| Link | Why it links | What anonymity would need, in principle |
|---|---|---|
| Billing and account signup | Payment and KYC name a person | Privacy-preserving payment designs exist, in the literature and in practice (some providers accept cash). bpb does not cover them; this is a scope choice, not an impossibility |
| The anchor | One hub that talks to every box is a correlation point by construction | No hub. Each identity gets its own compartment, and nothing crosses between them |
| The relay | It's billed too, and it's long-lived. Every box talks to it | No stable endpoint shared across runs or identities |
| Heartbeat and event stream | A fixed 10-second signed beacon is ideal timing-correlation material | Constant-rate cover traffic through a mixnet. Events ride inside the cover stream, so a gap still means something and timing reveals nothing |
| Operator activity hours | Provisioning and alerts follow your waking hours and time zone | Scheduling decoupled from when you are awake |
| The public flake | Configuration, closure hash and sensor footprint identify bpb, and the user count is tiny | Look like a common, boring configuration. This directly fights hardening |
| Reused keys and accounts | The same SSH CA, provider account or honey-credential account across runs | Fresh, unrelated material per identity, generated in that compartment |
| Canary callbacks | Every callback lands on infrastructure someone registered | Callbacks to a receiver inside the same compartment, reached over an anonymity network. A trip then links to that compartment, not to you. Honey credentials minted at third parties stay a link |
| The ledger | A signed record of every run | No cross-run record at all |
| Alerts to your phone | Ties events to a device that is already tied to you | No alert path back to a personal device |
| The repo | Git authorship, hosting account, commit times | Kept apart from any identity that operates boxes |
| The data itself | The content can identify who would have it | Out of scope for infrastructure |

### Tension with bpb's own principles

Anonymity would reshape the control plane, not the box. How each core idea fares:

| bpb principle | Under anonymity |
|---|---|
| 11. Keep a ledger | Survives only per compartment, or not at all. Never across identities |
| 7. Silence is a signal | Survives with constant-rate cover traffic. A plain beacon does not |
| 8. Canaries fill the blind spots | Survives with compartment-local receivers. Detection is slower and costs more |
| 6. Evidence must leave the box at event time | Survives inside the cover stream, at the cost of bandwidth and latency |
| 4. One job, one box, one identity | Keeps, but has to extend to everything upstream of the box, including the anchor |
| 5. Architecture substitutes for trust | Keeps, and becomes the whole job |

### Verdict

An anonymous bpb is a different control plane on the same box.

- **What carries over:** the box side, unchanged: tmpfs root, LUKS, lockdown, sensors and the airlock.
- **What changes:**
  - The single anchor splits into per-identity compartments that share nothing.
  - The ledger lives per compartment or not at all.
  - The heartbeat becomes constant-rate cover traffic through a mixnet.
  - Canaries report to receivers inside their own compartment.
  - The configuration tries to look common rather than distinctive.
- **What it costs:** latency, bandwidth, money and detection speed. The detection doesn't disappear.
- **What it hinges on:**
  - Payment, which bpb does not cover by choice.
  - How long each compartment lives. A short-lived compartment with a narrow purpose exposes far less than one person operating for years, where one slip is close to certain.

It stays a non-goal because it is a second project's worth of control-plane work, not because it is impossible.

---

[^luksdump]: cryptsetup-luksDump(8) man page (LUKS2 header fields: version, cipher, keyslots, UUID, digest). https://man.archlinux.org/man/cryptsetup-luksDump.8.en

[^nist80088]: NIST SP 800-88 Rev. 1, Guidelines for Media Sanitization (Clear / Purge / Destroy; cryptographic erase), via the NIST ITL bulletin. https://csrc.nist.gov/files/pubs/shared/itlb/itlbul2015-02.pdf

[^ssderase]: Wei, Grupp, Spada and Swanson, "Reliably Erasing Data From Flash-Based Solid State Drives" (USENIX FAST '11), as reported by InfoWorld. https://www.infoworld.com/article/2623513/flash-based-solid-state-drives-nearly-impossible-to-erase.html

[^coldboot]: Halderman et al., "Lest We Remember: Cold-Boot Attacks on Encryption Keys" (USENIX Security 2008), and Kicksecure's Cold Boot Attack Defense page summarizing 3MDEB's 2024–25 DDR4/DDR5 retention findings. https://static.usenix.org/event/sec08/tech/full_papers/halderman/halderman_html/ and https://www.kicksecure.com/wiki/Cold_Boot_Attack_Defense

[^scrambler2017]: Bauer, Gruhn and Freiling, "Cold Boot Attacks are Still Hot: Security Analysis of Memory Scramblers in Modern Processors" (2017), as discussed on the Whonix forums. https://forums.whonix.org/t/cold-boot-attacks-are-still-hot-security-analysis-of-memory-scramblers-in-modern-processors-feb-2017/3913

[^vmiforensics]: Cox et al., cloud VM memory acquisition through virtual machine introspection (ACM CODASPY 2018). https://www.cs.yale.edu/homes/abhishek/gcox-codaspy18.pdf

[^fjta]: "FJTA: Forensic Journal Timeline Analysis" and related journal-based timeline work (CODE BLUE 2025), on rebuilding file activity from ext4/XFS journals independent of MACB timestamps. https://codeblue.jp/en/program/time-table/day2-t3-04

[^snapshot]: (primary) DigitalOcean docs, "How to Destroy Droplets": destroying a Droplet does not destroy its automated backups, and its snapshots survive unless selected for deletion. https://docs.digitalocean.com/products/droplets/how-to/destroy/

[^murdoch]: Steven J. Murdoch and George Danezis, "Low-Cost Traffic Analysis of Tor", IEEE S&P 2005. https://www.murdoch.is/papers/oakland05torta.pdf

[^johnson]: Aaron Johnson et al., "Users Get Routed: Traffic Correlation on Tor by Realistic Adversaries", ACM CCS 2013. https://www.ohmygodel.com/publications/usersrouted-ccs13.pdf

[^ulbricht]: NBC News, "End of Silk Road: Slip-ups in building Internet drug market led to Ross Ulbricht, court documents say" (October 2013). https://www.nbcnews.com/news/world/end-silk-road-slip-ups-building-internet-drug-market-led-flna8C11326181

[^sabu]: The Register, LulzSec takedown analysis (March 2012), which reports both the single-IRC-login account and Monsegur's dispute of it. https://www.theregister.com/2012/03/07/lulzsec_takedown_analysis/

[^sr2]: Help Net Security, "Silk Road 2.0 shut down, operator arrested, charged" (November 2014): an undercover agent on the support staff, and a server leased under a personal email address. https://www.helpnetsecurity.com/2014/11/07/silk-road-20-shut-down-operator-arrested-charged/

[^alphabay]: (secondary) Wikipedia, AlphaBay (court-document account of the operator's arrest while logged into his unencrypted laptop performing an administrative reboot of an AlphaBay server; operational-security errors including a personal email address). https://en.wikipedia.org/wiki/AlphaBay

[^kingdom]: The Record, "Kingdom Market administrator gets 16-year sentence": crypto traced to wallets in the administrator's name, and one IP address tying a Reddit account to his personal accounts. https://therecord.media/kingdom-market-administrator-gets-16-year-sentence

[^incognito]: Cointelegraph, "Incognito Market founder arrested, charged in New York" (May 2024): bitcoin swapped to Monero, then deposited at an exchange that held his ID. https://cointelegraph.com/news/incognito-market-founder-arrested-charged-new-york

[^monopoly]: (secondary) Security Affairs, on charges against the alleged Monopoly Market administrator (2023): the seized server held vendor communications, commission invoices and cryptocurrency payment records. https://securityaffairs.com/147849/cyber-crime/monopoly-market-admin-charged.html

[^oxymonster]: Security Affairs, "Dream Market dark web drug dealer OxyMonster arrested on way to beard contest" (September 2017): tip jar paid to his own wallets, writing-style comparison, PGP key found on his laptop at the border. https://securityaffairs.com/63689/cyber-crime/market-dark-oxymonster-arrested.html

[^bohemia]: The Record, on the Bohemia/Cannabia administrators' arrests (October 2024): exit-scam funds traced, devices with wallet keys seized at Schiphol airport. https://therecord.media/bohemia-cannabia-dark-web-market-arrests

[^wsm]: TechCrunch, "How German and US authorities took down the owners of darknet drug emporium Wall Street Market" (2019): VPN provider records correlated with a suspect's home connection. https://techcrunch.com/?p=1822049
