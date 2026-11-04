# bpb: bulletproof box

A NixOS flake that turns any machine you can SSH into (a rented GPU box, a VPS, a closet server, a QEMU VM) into a hardened, attested, self-reporting, burnable compute node. You deploy it with `nixos-anywhere`, it proves what it is running, it screams when touched, and it can be destroyed so thoroughly that a seized disk is noise.

## Status

**This repository is a design document, not working software.** Nothing here is built, tested, or audited yet — every milestone in the [roadmap](docs/roadmap.md) is unimplemented. It is a learning project, and its one rule is that every control must ship with a test that proves it fires. Read it as a plan and a threat model, not as something to deploy.

## The idea

Rent a machine from a provider you do not trust, and assume its operator has root from the first second. bpb is built around that assumption:

- **Deploy** into a hardened, ephemeral node whose root filesystem is RAM and whose store is read-only and verity-checked.
- **Attest** what the box is actually running, so a tampered node is never trusted.
- **Report** tamper-evident evidence off-box, in real time, to an independent recorder that alerts when the stream stops.
- **Burn** the node so a later disk image yields only tokenized, decoy-padded noise.

It uses four roles: an **anchor** at home that holds every key, an untrusted **box** that does the work, a **relay** that is the anchor's only public face, and a **witness** that independently records what happened.

## What it does not do

A security project is only honest if it says where it stops. bpb does **not** hide who runs it, does **not** stop a provider with physical access to the memory bus, and concedes several adversaries outright. Read [limits](docs/limits.md) and the [threat model and non-goals](docs/threat-model.md) before trusting it with anything.

## Documentation

- [Threat model](docs/threat-model.md) — who the adversary is, the privilege ladder, and what bpb does not try to do.
- [Architecture](docs/architecture.md) — the four roles, the module map, and the box lifecycle.
- [Controls](docs/controls.md) — every defensive mechanism, by the idea it serves.
- [Testing](docs/testing.md) — red-team mode and the hostile-host lab.
- [Tiers](docs/tiers.md) — where a job runs (T0–T4) and which compute profiles have confidential hardware.
- [Operator](docs/operator.md) — the controls a person performs: identities, keys, backups, drills.
- [Limits](docs/limits.md) — forensic exposure, the honest ceiling, and operator anonymity.
- [Roadmap](docs/roadmap.md) — milestones and open questions.

Start with [the index](docs/index.md) for the core ideas and the full table of contents.

## License

Not yet chosen.
