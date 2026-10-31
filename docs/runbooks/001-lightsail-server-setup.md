# Herd OS — AWS Lightsail Server Setup and First SSH Connection

**Status:** Completed through first successful SSH login and host inspection  
**Date:** 2026-10-31  
**Environment:** AWS Lightsail, Canada (Central), Ubuntu 24.04 LTS  
**Scope:** Initial provisioning and SSH access. This document does **not** claim server hardening or OpenClaw deployment is complete.

## 1. Objective

Provision an always-on, remotely accessible Linux host for the Herd OS agent gateway. Use AWS CLI to understand the underlying resources before automating the process with infrastructure as code. Keep the application architecture portable to another VPS provider.

## 2. Selected infrastructure

| Setting | Value |
|---|---|
| Cloud provider | AWS Lightsail |
| AWS region | `ca-central-1` |
| Availability zone | `ca-central-1a` |
| Instance name | `herd-os-gateway` |
| Image / blueprint | `ubuntu_24_04` |
| Instance bundle | `small_3_0` |
| Advertised bundle price at selection | USD $12/month, excluding extras and taxes |
| SSH key-pair name in Lightsail | `herd-os` |
| Local private key | `~/.ssh/herd-os-lightsail` |
| Local public key | `~/.ssh/herd-os-lightsail.pub` |
| Initial public IP observed | `15.223.69.246` (not yet reserved/static) |
| Default login user | `ubuntu` |

**Security:** Never commit the private key or other credentials to Git. The IP address above is an observation, not a guaranteed permanent address.

## 3. Explore available AWS resources (Mac terminal)

We inspected Lightsail bundles, Ubuntu blueprints, and availability zones before provisioning:

```bash
aws lightsail get-bundles --region ca-central-1
aws lightsail get-blueprints --region ca-central-1
aws lightsail get-regions --region ca-central-1 --include-availability-zones
```

Selected the Ubuntu 24.04 blueprint, 2 GB bundle, and `ca-central-1a` availability zone. The AWS CLI must already be installed and authenticated; that earlier configuration is outside the verified scope of this runbook.

## 4. Generate the SSH key locally (Mac terminal)

We initially tried an Ed25519 key, then generated a 4096-bit RSA key during troubleshooting. The RSA key is the one successfully imported and used by the instance.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/herd-os-lightsail -C "herd-os-lightsail"
```

This creates:

- `~/.ssh/herd-os-lightsail` — **private key**, stays on the Mac.
- `~/.ssh/herd-os-lightsail.pub` — **public key**, imported into Lightsail.

Recommended local permissions, if needed:

```bash
chmod 600 ~/.ssh/herd-os-lightsail
chmod 644 ~/.ssh/herd-os-lightsail.pub
```

Check the public-key fingerprint without exposing the private key:

```bash
ssh-keygen -lf ~/.ssh/herd-os-lightsail.pub
```

## 5. Import the SSH public key into Lightsail (Mac terminal)

**Working command, verified successful:**

```bash
aws lightsail import-key-pair \
  --region ca-central-1 \
  --key-pair-name herd-os \
  --public-key-base64 "$(cat ~/.ssh/herd-os-lightsail.pub)"
```

The operation returned `status: Succeeded` and `resourceType: KeyPair`.

Verify:

```bash
aws lightsail get-key-pairs \
  --region ca-central-1 \
  --query 'keyPairs[].name' \
  --output table
```

Observed result: `herd-os`.

### Troubleshooting encountered

1. An initial Ed25519 import failed with `InvalidInputException: The format of this public key is not valid.` This alone does **not** prove that Ed25519 is universally unsupported.
2. Manually Base64-encoding the RSA `.pub` file and passing it as the argument also returned an invalid-format error.
3. `fileb://$HOME/.ssh/herd-os-lightsail.pub` failed with `ParamValidation`: the command parameter expected a Python `str`, but `fileb://` supplied `bytes`.
4. Passing the OpenSSH public-key file **as text** via `$(cat ...)` succeeded.

**Lesson:** CLI parameter naming, accepted key formats, and CLI serialization are separate concerns. For this environment, the exact successful command above is the verified method. Do not assume the parameter name implies you should Base64-encode the entire `.pub` file yourself.

## 6. Create the Lightsail instance (Mac terminal)

The initial attempt to name the instance `herd-os` failed with `Some names are already in use: herd-os`. `get-instance` returned `NotFoundException`; `get-instances` showed no instances, while `get-key-pairs` showed the `herd-os` key. We switched to a distinct instance name, `herd-os-gateway`, which succeeded. The precise underlying name-uniqueness rule was not independently established.

**Billable operation:**

```bash
aws lightsail create-instances \
  --region ca-central-1 \
  --instance-names herd-os-gateway \
  --availability-zone ca-central-1a \
  --blueprint-id ubuntu_24_04 \
  --bundle-id small_3_0 \
  --key-pair-name herd-os
```

AWS initially returned an operation with `status: Started`. This meant the provisioning request was accepted, not that the server was already ready.

Verify until `State` is `running`:

```bash
aws lightsail get-instance \
  --region ca-central-1 \
  --instance-name herd-os-gateway \
  --query 'instance.{Name:name,State:state.name,PublicIP:publicIpAddress,Blueprint:blueprintId,Bundle:bundleId,KeyPair:sshKeyName}' \
  --output table
```

**Observed:**

```text
Blueprint  ubuntu_24_04
Bundle     small_3_0
KeyPair    herd-os
Name       herd-os-gateway
PublicIP   15.223.69.246
State      running
```

## 7. Connect over SSH (Mac terminal)

```bash
ssh -i ~/.ssh/herd-os-lightsail ubuntu@15.223.69.246
```

Replace the IP with the current value from `get-instance` if it changes.

**Command anatomy:**

- `ssh`: opens an encrypted remote shell.
- `-i ~/.ssh/herd-os-lightsail`: selects the local **private** key.
- `ubuntu`: default user for the selected Ubuntu image.
- `@15.223.69.246`: destination server's public IP.

**Authentication concept:** AWS installs/authorizes the imported **public** key on the instance; the Mac proves possession of the corresponding **private** key during SSH authentication. The private key is not uploaded to the server.

On first connection, SSH may show an unknown-host-key prompt. The **server host key** is distinct from the **user authentication key**. For high-assurance verification, obtain the server's host-key fingerprint through a trusted channel and compare it before accepting. SSH records accepted host keys in `~/.ssh/known_hosts` on the Mac.

A successful connection was confirmed by the remote shell prompt:

```text
ubuntu@ip-172-26-11-154:~$
```

## 8. Inspect the server (Ubuntu remote shell)

```bash
whoami
hostname
cat /etc/os-release
free -h
df -h
```

**Observed results:**

| Check | Result |
|---|---|
| Login user | `ubuntu` |
| Hostname | `ip-172-26-11-154` |
| OS | Ubuntu 24.04.4 LTS (Noble Numbat) |
| RAM | 1.9 GiB total; ~1.5 GiB available at inspection |
| Swap | 0 B configured |
| Root disk | 58 GiB total; ~56 GiB available; 4% used |

The private-looking hostname reflects the instance's internal network identity; the public IP is used for access from the Mac. A 2 GB host is a reasonable starting point for a lightweight gateway that calls hosted models, **not** for hosting a large LLM locally.

## 9. Next steps — NOT YET COMPLETED

- [ ] Apply OS updates (`sudo apt update` and review/install upgrades).
- [ ] Configure unattended security updates.
- [ ] Inspect Lightsail network firewall and Ubuntu UFW rules **before** modifying SSH access.
- [ ] Decide whether to add swap and configure it.
- [ ] Reserve and attach a static Lightsail IP; update SSH destination as necessary.
- [ ] Establish private administration (e.g., Tailscale) and avoid exposing the agent gateway publicly.
- [ ] Establish backups/snapshots, cost monitoring, and recovery procedure.
- [ ] Install Docker and deploy OpenClaw with secrets managed securely.
- [ ] Connect Telegram and test an end-to-end workflow.
- [ ] Add health checks, structured logs, task outcomes, and alerting.
- [ ] Convert the manual provisioning steps into reproducible IaC after validating the design.

## 10. Useful operational commands

Run these on the **Mac**, not inside the Ubuntu SSH session:

```bash
# Read-only instance status
aws lightsail get-instance \
  --region ca-central-1 \
  --instance-name herd-os-gateway

# Read-only list of instance firewall port rules
aws lightsail get-instance-port-states \
  --region ca-central-1 \
  --instance-name herd-os-gateway

# Reconnect
ssh -i ~/.ssh/herd-os-lightsail ubuntu@15.223.69.246
```

To leave the remote shell without shutting down the server:

```bash
exit
```

**Cost reminder:** Closing SSH does not stop the Lightsail instance or its billing. Stopping a Lightsail instance does not necessarily stop its instance charges; delete resources only when intentionally decommissioning them, after securing data and backups.
