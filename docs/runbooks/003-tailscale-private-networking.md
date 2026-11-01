# Runbook 003 — Tailscale Private Networking

**Project:** Personal AI OS / Herd OS  
**Date:** 2026-11-01  
**Status:** Tailnet enrollment verified; private SSH and public SSH shutdown pending.

## Objective

Connect the developer Mac and AWS Lightsail Ubuntu gateway over a private encrypted Tailscale network, then validate SSH through that network before restricting public SSH access.

## Existing infrastructure

| Item | Value |
| --- | --- |
| AWS service | Lightsail |
| Region | `ca-central-1` |
| Instance | `herd-os-gateway` |
| Ubuntu | 24.04.4 LTS |
| Lightsail public IP observed during setup | `15.223.69.246` (not yet reserved as static) |
| SSH username | `ubuntu` |
| Local SSH private key | `~/.ssh/herd-os-lightsail` |
| Mac Tailscale IP | `100.85.130.7` |
| Server Tailscale IP | `100.108.177.8` |

## What was completed

1. Installed Tailscale on the Ubuntu server using the official Linux installer, after downloading and inspecting the script.
2. Authenticated the server to the tailnet using `sudo tailscale up`.
3. Installed and signed into the Tailscale macOS app with the same account.
4. Confirmed both devices appear in the Tailscale device list.
5. Confirmed from the server:

```text
100.108.177.8  ip-172-26-11-154     ...  linux  -
100.85.130.7   joshuas-macbook-pro  ...  macOS  -
```

The addresses above are Tailscale addresses, not public internet IPs. Tailscale device enrollment does not by itself prove that SSH over the tailnet works.

## Verify private SSH — NEXT STEP

Keep the existing public-IP SSH session open. On the **Mac**, open a second terminal and run:

```bash
ssh -i ~/.ssh/herd-os-lightsail ubuntu@100.108.177.8
```

The first connection to the Tailscale IP may prompt for a host-key trust decision. Compare the fingerprint against the already trusted server entry (or a fingerprint retrieved over the existing trusted SSH connection) before accepting. On the server, inspect fingerprints with:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

After connecting through the Tailscale IP, verify:

```bash
whoami
hostname
tailscale status
```

Expected user: `ubuntu`; hostname: `ip-172-26-11-154`.

## Restrict public SSH — ONLY AFTER PRIVATE SSH WORKS

**Do not close port 22 yet.** First ensure the Mac can reconnect using the Tailscale IP, ideally in a fresh session. Keep a recovery route available via the Lightsail browser-based SSH console.

From the **Mac** (AWS CLI configured locally), inspect the public firewall:

```bash
aws lightsail get-instance-port-states \
  --region ca-central-1 \
  --instance-name herd-os-gateway \
  --output table
```

When ready, remove public TCP port 22:

```bash
aws lightsail close-instance-public-ports \
  --region ca-central-1 \
  --instance-name herd-os-gateway \
  --port-info '{"fromPort":22,"toPort":22,"protocol":"tcp"}'
```

Then verify the firewall again and confirm **a new** SSH connection to `100.108.177.8` succeeds. Do not assume an already-established SSH session is sufficient proof.

### Networking note

Tailscale typically establishes outbound encrypted connections and can use relay servers when direct peer-to-peer connectivity is unavailable. Closing Lightsail's inbound port 22 does not normally prevent Tailscale connectivity. Host firewall rules and tailnet access policies must still be considered separately.

## Current security posture

- Lightsail TCP port 80 was closed earlier.
- Lightsail TCP port 22 is currently open to `0.0.0.0/0` and `::/0`.
- Ubuntu UFW was observed **inactive**.
- SSH key authentication has been demonstrated.
- Tailscale device enrollment has been demonstrated.
- Private SSH access **has not yet been demonstrated**.

## Next tasks

1. Verify SSH to `100.108.177.8` from a fresh Mac terminal.
2. Check that the Lightsail browser SSH console provides a usable recovery option.
3. Close public port 22 and test private SSH again.
4. Decide on Tailscale device key expiry and access controls.
5. Continue with Docker and OpenClaw deployment; keep the agent gateway bound to localhost or a private interface, not publicly exposed.

## References

- Tailscale Linux installation: https://tailscale.com/kb/1031/install-linux
- Tailscale macOS installation: https://tailscale.com/kb/1016/install-mac
- Tailscale SSH documentation: https://tailscale.com/kb/1193/tailscale-ssh

> Note: This setup uses **regular OpenSSH transported over Tailscale**, not Tailscale SSH's optional identity-managed SSH feature.
