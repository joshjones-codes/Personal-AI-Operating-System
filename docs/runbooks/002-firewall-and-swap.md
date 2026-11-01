# Herd OS — Firewall Inspection and Swap Configuration

**Status:** Firewall inspection, HTTP closure, and swap setup completed; private networking and SSH restriction pending  
**Recorded:** 2026-11-01  
**Host:** `herd-os-gateway` · Ubuntu 24.04.4 LTS · AWS Lightsail `ca-central-1` · `small_3_0` (2 GB RAM)

## Objective

Reduce unnecessary network exposure and add a small memory-pressure safety net before deploying the agent gateway. Record what was actually performed and what remains to be verified.

## 1. Understand the two firewall layers

Traffic from the internet first passes through the **AWS Lightsail network firewall**, then reaches the Ubuntu host, where a host firewall such as **UFW** can apply additional rules.

- **Port 22/TCP:** SSH remote administration.
- **Port 80/TCP:** Unencrypted HTTP; not needed for the current deployment.
- **`0.0.0.0/0`:** Any IPv4 source address.
- **`::/0`:** Any IPv6 source address.

An open port in the network firewall allows packets to reach the host; it does **not** mean a process is listening or that a connection is authenticated.

### Inspect Ubuntu UFW (run on the server)

```bash
sudo ufw status verbose
```

**Observed:** `Status: inactive`. We did **not** enable or configure UFW during this stage. The Lightsail firewall remains the active network-level restriction.

### Inspect Lightsail firewall (run on the Mac)

```bash
aws lightsail get-instance-port-states \
  --region ca-central-1 \
  --instance-name herd-os-gateway \
  --output table
```

**Initially observed:** TCP ports 22 and 80 open to `0.0.0.0/0` and `::/0`.

> **Execution context matters:** The AWS CLI was configured on the Mac, not on the remote Ubuntu instance. Running `aws` inside the SSH session produced `Command 'aws' not found`. There is no need to install the AWS CLI on the server for these operations.

## 2. Close unnecessary public HTTP access

**Run on the Mac:**

```bash
aws lightsail close-instance-public-ports \
  --region ca-central-1 \
  --instance-name herd-os-gateway \
  --port-info '{"fromPort":80,"toPort":80,"protocol":"tcp"}'
```

Re-ran `get-instance-port-states` and **verified** that only TCP port **22** remained open, with source ranges `0.0.0.0/0` and `::/0`.

**Current exposure:** Public SSH is still allowed from any address. Do not close it until private connectivity is installed and verified from a second terminal/session.

## 3. Configure a 2 GiB swap file

**Run on the Ubuntu server:**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

- `fallocate` reserves disk space at `/swapfile`.
- `chmod 600` limits access to the owner (root).
- `mkswap` prepares the file as Linux swap space.
- `swapon` activates it immediately.

**Verified:**

```bash
free -h
```

Output showed approximately **1.9 GiB RAM**, **2.0 GiB swap**, and **0 B swap used** at the time of inspection.

### Persist swap after reboot

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

This adds a boot-time swap entry. The entry was written successfully, but **persistence has not yet been tested by rebooting**. Avoid rerunning this append command without checking whether the line already exists.

### Optional checks

```bash
ls -lh /swapfile
swapon --show
grep -n '^/swapfile ' /etc/fstab
```

**How swap works:** Swap is a **file on the instance's disk**, not additional physical RAM. Linux may move less-active memory pages there under memory pressure. Disk-backed swap is substantially slower than RAM and is a safety net, not a substitute for resizing the instance if memory usage is consistently high.

## 4. Security and operational follow-ups

1. **Install Tailscale** and verify private SSH access from the Mac while the existing public SSH session remains available.
2. **Restrict/close public port 22** in the Lightsail firewall only after confirming private connectivity and a recovery path.
3. **Verify SSH daemon settings**, especially password authentication, before relying on SSH-key-only access. This has not yet been checked.
4. **Confirm OS updates and unattended-upgrades** status; earlier instructions were provided, but completion was not verified in this session.
5. **Reserve a static IP** if required for the chosen access/deployment design; current public IP is not reserved.
6. **Consider host firewall (UFW)** as defense in depth, coordinated with Tailscale rules to avoid lockout.
7. **Add monitoring/backups** and document recovery procedures before running autonomous agents.

## 5. Next runbook

`003-tailscale-private-access.md` — Install Tailscale on the server and Mac, authenticate devices, test connectivity, then remove public SSH exposure after verification.

## Principles learned

- **Cloud firewall vs. host firewall:** independent enforcement points.
- **Network reachability vs. authentication:** an allowed SSH connection still requires valid credentials.
- **Local vs. remote shell:** commands run wherever the current terminal is connected.
- **RAM vs. swap:** swap is disk-backed virtual-memory overflow, not equivalent to RAM.
- **Verification matters:** record command output and distinguish configured from tested (especially reboot persistence).
