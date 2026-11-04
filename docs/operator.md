# Operator

The controls a person performs. They aren't code, but they are still controls.

- One identity per provider: alias email, separate password vault, separate browser profile. Payment is the hard link. Decide per provider whether KYC is acceptable.
- Snapshot each provider's ToS, privacy policy, and DPA at signup. Diffing them at renewal is optional: it is one of the chores most likely to rot, and a skipped diff costs less than the others.
- **A provider account is a root credential for every box it owns.** Through the web console it can open a serial or VNC session, reboot into rescue mode, snapshot disks and delete evidence, none of which touches sshd. One documented self-hoster compromise went exactly this way: no SSH session at all, a failed login on `tty1`, a reboot, then a new sudo user created from the console. The most likely cause was the provider account or the provider itself.[^vpsconsole] Credential theft, not network attack, starts most breaches.[^homestack] So:
  - every provider account uses hardware-key MFA, never SMS or TOTP alone
  - API tokens are scoped as narrowly as the provider allows (per project, and read-only where read is enough), stored encrypted on the anchor under the hardware token, never placed on a box, and rotated whenever the provider identity rotates
  - the provider's login and audit notifications go to an inbox the operator actually reads, and a console-session notification during a run is treated as a tripwire
- Anchor keys live on two enrolled hardware tokens, one key per role (see Anchor), including the two SSH CAs. The operator's own SSH keys are FIDO hardware keys (`ssh-keygen -t ed25519-sk`), which OpenSSH supports natively. That is far simpler than GPG-based hardware-key setups.[^guidethread][^openssh82] Off-site, at a second location: the spare token's PIN and a paper recovery key (or Shamir shares of it), plus the written recovery steps.
- Quarterly restore drill of the anchor, using only the spare token, on blank hardware, from the newest backup. An anchor you cannot rebuild is a single point of total loss, and a drill that uses the main token never tests the most likely loss.
- Run `gitleaks` on this repo and your dotfiles. The flake is public-safe only if secrets live in agenix/sops and canary layouts are generated, not committed.
  - Scan the full history (`gitleaks git --log-opts="--all"`)[^gitleaks], not just the working tree: a secret deleted in a later commit is still in history. Run it as a pre-commit hook too, so leaks are stopped before they are pushed.
  - On a hit, rotate first, then clean history (`git-filter-repo`). Cleaning alone is not a fix: anyone who cloned or cached the repo already has the secret.

---

[^vpsconsole]: r/selfhosted, "I thought my VPS was hardened, but it was compromised and I can't figure out how. Please help!" (April 2026, 149 comments), with the owner's logs and the community diagnosis of provider-console access. Read via Arctic Shift. https://www.reddit.com/r/selfhosted/comments/1se8ara/

[^homestack]: r/selfhosted, "My HomeServer Security Hardening Stack – Feedback & Review" (January 2026), comments on credential-led breaches and on rarely used paths failing quietly. Read via Arctic Shift. https://www.reddit.com/r/selfhosted/comments/1qhb8oe/

[^guidethread]: r/selfhosted, "[Guide] Securing A Linux Server" (August 2024), comments on Docker bypassing UFW and on native FIDO SSH keys. Read via Arctic Shift. https://www.reddit.com/r/selfhosted/comments/1f3y16m/guide_securing_a_linux_server/

[^openssh82]: OpenSSH 8.2 release notes (FIDO/U2F hardware authenticator support, `ecdsa-sk` and `ed25519-sk` key types). https://www.openssh.org/txt/release-8.2

[^gitleaks]: OneUptime, "How to Implement Secret Scanning with gitleaks" (January 2026). https://oneuptime.com/blog/post/2026-01-25-secret-scanning-gitleaks/markdown
