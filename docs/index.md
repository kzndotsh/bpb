# bpb: bulletproof box

A NixOS flake that turns any machine you can SSH into (a rented GPU box, a VPS, a closet server, a QEMU VM) into a hardened, attested, self-reporting, burnable compute node. You deploy it with `nixos-anywhere`, it proves what it is running, it screams when touched, and it can be destroyed so thoroughly that a seized disk is noise.

It is a learning project. Every control ships with a test that proves it fires.

## The core ideas, distilled

These came out of the original theorycrafting. Everything below is an implementation of one of them.

1. **Threat model before controls.** Name the adversary and its privilege rung. Untargeted paranoia burns out or locks you out of your own data.
2. **Sanitize beats encrypt.** Encryption hides content. It does not hide filenames, sizes, timing, counts, or stable hashes. Metadata survives encryption.
3. **Minimize before you send.** The only data that cannot leak is data that never left.
4. **One job, one box, one identity, then burn it.** Nothing remote is long-lived.
5. **Architecture substitutes for trust.** Policies are revocable promises. Design so a broken promise costs you a bag of anonymous bytes.
6. **Detection only works below the adversary's privilege or out of band.** Root vs. root is best effort. Hypervisor and BMC are blind spots. So evidence must leave the box at event time.
7. **Silence is a signal.** A box that stops reporting is a finding, not a gap.
8. **Canaries fill the blind spots.** Decoys that phone home cover what sensors cannot see, including leak attribution after the fact.
9. **Shrink the plaintext window.** An unlocked volume surrenders to any image taken while mounted. Swap, caches, logs, and history are plaintext copies.
10. **Never trust the return path.** A hostile box can forge output. Validate it at the boundary.
11. **Keep a ledger.** "Did X leak?" is only answerable with a record of what left, when, and where.
12. **You are an adversary too.** Lost keys and complexity collapse destroy more data than attackers. Escrow and restore drills are controls.
13. **Know the ceiling.** A competent host operator is undetectable from inside. Price it in. Make a successful snoop boring.

## The documents

- [Threat model](threat-model.md) — Who the adversary is, the privilege ladder, and what bpb does not try to do.
- [Architecture](architecture.md) — The four roles, the module map, and the box lifecycle.
- [Controls](controls.md) — Every defensive mechanism, by the idea it serves.
- [Testing](testing.md) — Red-team mode and the hostile-host lab.
- [Tiers](tiers.md) — Where a job runs (T0–T4) and which compute profiles have confidential hardware.
- [Operator](operator.md) — The controls a person performs: identities, keys, backups, drills.
- [Limits](limits.md) — Forensic exposure, the honest ceiling, and operator anonymity.
- [Roadmap](roadmap.md) — Milestones and open questions.
- [References](references.md) — Every source cited across the docs.
