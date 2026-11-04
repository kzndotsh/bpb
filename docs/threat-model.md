# Threat model

## Topology: four roles

```
         ┌──────────────┐  WG  ┌───────┐  WG peer 1: job traffic  ┌──────────────┐
         │    ANCHOR    │◄────►│ RELAY │◄────────────────────────►│     BOX      │
         │ (home, trusted)     │ (VPS) │  (ciphertext forwarded   │ (untrusted)  │
         │ keys, tang,  │      └───────┘   to the anchor)         │ tmpfs root,  │
         │ airlock,     │                                         │ LUKS work vol│
         │ ledger       │                                         │ sensors      │
         └──────┬───────┘                                         └──────┬───────┘
     heartbeat, │ ▲ signed verdicts,                     WG peer 2:      │ events +
     registrations ▼ receipts                            evidence only   │ heartbeat
                │        ┌──────────────┐                (direct, not    │
                └───────►│   WITNESS    │◄───────────────via anchor)─────┘
                         │ (3rd vantage)│  append-only, hash-chained,
                         │ Pi / tiny VPS│  enforces its own retention
                         └──────────────┘  alerts → phone, no Apple/Google push
```

- **Anchor** is the system of record. It holds every key, runs the Tang servers, the airlock (sanitizer), the ledger, and the provisioning CLI. It never runs untrusted workloads. Because it holds everything, it is the most valuable target in the system and is hardened and watched like one (see Anchor). It is the only party that can act (release Tang, revoke, freeze, burn), but it acts on the witness's verdicts, not only its own view.
- **Relay** is a small VPS that is the anchor's only public face. It forwards the box's job-traffic WireGuard UDP inside its own tunnel to the anchor and never terminates the box's tunnel, so it sees only ciphertext. Provisioning traffic (`nixos-anywhere`, provider API calls) also leaves through it. Hostile hosts learn the relay's address, not the anchor's, so they cannot probe or attack the anchor directly. It hides nothing from a provider that can see your billing (see Non-goals).
- **Box** is the target. Assume its operator has root from second zero. It holds no long-lived secrets and no identity.
- **Witness** is a separate, boring machine. It exists so neither a compromised box nor a compromised anchor can quietly rewrite history, which only works if box evidence does not pass through the anchor. So:
  - **Evidence has its own path.** The box has a second WireGuard peer, the witness, reached directly at the witness's own address (its own nftables and provider-firewall rule), carrying only events and heartbeats. Neither the anchor nor the relay is on that path, so neither can drop, delay or read it. An anchor or relay outage no longer silences the box's evidence.
  - **The anchor cannot mint box evidence.** The box's event-signing key is generated on the box at boot and bound into its attestation (see Box lifecycle). The anchor only *registers* the attested public key with the witness, and that registration is itself a logged, signed witness event. A compromised anchor that registers a key of its own does it on the record.
  - **The witness decides; the anchor acts.** The witness sends signed verdicts (stream healthy at seq N, gap, tripwire) to the anchor over their own WireGuard link. The anchor releases Tang at arm only against a fresh witness "stream healthy" receipt, so "no witness, no job" is enforced by the anchor, not by box code that root can ignore.
  - **The witness owns its history.** It enforces its own retention clock and never deletes a commitment. The anchor has no delete authority over it (see Ledger).
  - **Anchor and witness watch each other.** Each alerts on the other's silence, so a compromised witness that suppresses its own alerts is still caught by the anchor's watch of it, and vice versa.

## Threat model and privilege ladder

| Rung | Adversary | Can bpb detect it? | Primary control |
|---|---|---|---|
| 0 | Network observer (ISP, transit) | No | WireGuard only, DNS through tunnel. Content is hidden; endpoints, sizes and timing are not |
| 1 | Unprivileged process / other tenant | Yes | auditd, fanotify, egress deny |
| 2 | Root on the box, sloppy | Mostly | LSM-hook sensors, io_uring off, closure verify, kernel lockdown, off-box logs |
| 3 | Root on the box, careful | Sometimes | Heartbeat gaps, canaries, egress via anchor |
| 4 | Hypervisor / host operator | No (from inside) | Tokenized data, ephemerality, per-box canaries, anchor-made keys, SEV-SNP/TDX tier. The host also owns the box's clock and entropy |
| 5 | BMC / IPMI / physical | No | LUKS + Tang (disk dead off-network), cryptoshred. On metal the BMC may already be backdoored by a previous tenant |
| 6 | Legal process on the provider | No | Nothing worth handing over; ledger on your side. Servers get imaged silently and more than once while still running (Genesis Market's backend was copied twice, in 2020 and 2022, before the takedown;[^genesis] Silk Road's server was imaged in Iceland[^silkroad]) |
| 7 | Supply chain | Yes, at build | Pinned flake inputs, hash-locked weights/images, safetensors only, independent rebuild, patched Nix on the anchor |
| 8 | You | Yes, by drill | Escrow, restore drills, reversible panic before irreversible burn |
| 9 | Anchor seized or compromised (theft, raid, remote exploit) | Yes, via the witness | Anchor FDE behind two enrolled tokens, one key per role, per-run secrets never backed up, anchor sensors + mutual heartbeat with the witness, an evidence path that bypasses the anchor, witness-enforced ledger retention |

## Non-goals

Writing these down is part of principle 13. A plan that does not say what it cannot do gets trusted for things it cannot do.

- **Operator anonymity.** bpb keeps your data from a host that snoops. It does not hide who runs it from an adversary who can compel providers, ISPs, payment processors and phone vendors. Against that adversary the design points at the operator: billing, the public flake (any box that behaves like bpb is probably you, because the set of people running it is tiny), the ledger, the infrastructure canaries call back to, and the timing of the heartbeat. Fixing that would take a different architecture, not more controls, and it is out of scope. "Operator anonymity, in theory" lays out why.
- **Deniability.** LUKS headers, Tang keys and the ledger prove that encrypted data and runs exist. Nothing here lets you credibly claim otherwise.
- **Resisting legal process.** Burn is a privacy control against hosts that snoop. It is not a response to legal process: destroying data covered by a preservation order or legal hold can itself be a crime. Burn is never automatic in bpb, and `bpb hold` makes it refuse to run at all for every box until `bpb unhold`; boxes also stop powering themselves off on prolonged silence.

---

[^genesis]: CyberScoop, on the FBI's copies of Genesis Market's backend server (December 2020 and 2022) and the users identified from them (April 2023). https://cyberscoop.com/genesis-market-fbi-users-identified-arrests/

[^silkroad]: Help Net Security, "FBI's account of locating Silk Road's server disputed by researchers" (September 2014), covering the Iceland server image and the dispute over the CAPTCHA-leak account. https://www.helpnetsecurity.com/2014/09/09/fbis-account-of-locating-silk-roads-server-disputed-by-researchers/
