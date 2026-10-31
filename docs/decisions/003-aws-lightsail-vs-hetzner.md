# ADR-003: AWS Lightsail for MVP hosting, with Hetzner portability

- **Status:** Accepted for MVP; cost and sizing provisional
- **Date:** 2026-10-31
- **Decision owners:** Herd project

## Context

OpenClaw needs an always-on Linux host. Hetzner Cloud is typically attractive for inexpensive virtual machines; AWS provides broader infrastructure services and directly supports the user's goal of deepening AWS/backend/platform expertise. The initial workload is a gateway plus light integrations; model inference is expected to happen via external providers, not on the VPS.

## Decision

Start on **AWS Lightsail (Ubuntu Linux)**, initially targeting a **2 GB RAM class** instance for a lightweight gateway. Measure actual memory/CPU use and upgrade if necessary. Do not add AWS-specific managed services until a concrete requirement justifies them. **Preserve the option to migrate to Hetzner Cloud** or another Linux VPS provider.

## Why AWS for this project

1. Hands-on AWS deployment and operations are career-relevant objectives, not merely incidental hosting.
2. Lightsail offers a simpler entry point and more predictable instance bundles than an immediately complex EC2/VPC design.
3. We can later explore IAM, CloudWatch, S3, SQS, and CDK when they solve real problems.
4. The modest extra hosting cost may be justified by learning value.

## Why not Hetzner first

Hetzner can offer more RAM/CPU per dollar. If the only goal were the cheapest always-on VPS, it could be preferable. We are explicitly prioritizing AWS ecosystem practice for the MVP. **Hetzner is not inherently less capable of running the Gateway.**

## Portability requirements

- Keep runtime configuration in version-controlled templates and environment variables; never commit secrets.
- Prefer a reproducible deployment (Docker Compose or equivalent systemd-based install) and automated bootstrap scripts.
- Keep persistent application data in clearly identified volumes/directories with documented backup/restore procedures.
- Avoid embedding AWS-specific APIs in core agent logic; use adapters for S3/SQS/etc. if introduced.
- Keep DNS independent of the compute instance and use low TTL during planned migration.
- Maintain a migration runbook and a clean restore test.

## Proposed migration: AWS -> Hetzner

1. Provision a Hetzner Linux VM with appropriate RAM/storage and network policy.
2. Apply the same bootstrap/deployment configuration and install matching runtime versions.
3. Restore encrypted application data and secrets through a secure channel.
4. Reconnect messaging channels, OAuth integrations, and model credentials as required.
5. Validate tasks, persistent state, approvals, and notifications on the new host.
6. Switch DNS/webhook endpoints or channel connections; monitor; retain the old host briefly for rollback.
7. Decommission AWS resources only after validation and backup confirmation.

**Migration caveats:** OAuth callbacks, IP allowlists, provider-specific volumes, public endpoints, and AWS-managed services may require additional work. If the Gateway uses outbound polling rather than a public webhook, the cutover may be simpler. Never run two active consumers against a single exclusive messaging session without understanding their behavior.

## Costs and constraints

- Use current official AWS/Hetzner calculators before purchase; prices, taxes, transfer charges, IPv4 fees, and introductory credits can change.
- Set AWS Budgets alerts before deployment. Alerts are not hard caps.
- API model usage is separate from VPS charges; subscription compatibility must be verified.
- A 2 GB VM is an initial hypothesis, not a guaranteed capacity for browser automation or multiple concurrent coding workers. Heavy execution may need 4–8 GB or separate workers.

## MVP acceptance criteria

- Gateway runs on Lightsail with persistent state and restricted administration access.
- A documented deployment can recreate the environment on a fresh Linux host.
- Backups can be restored and a representative task succeeds after restore.
- Monthly infrastructure spend and agent usage are visible.

## Revisit if

- Hetzner savings become material relative to learning value.
- The workload needs compute/memory that is uneconomical on Lightsail.
- AWS-specific services or compliance requirements justify EC2/ECS instead.
- Provider reliability or networking constraints materially affect availability.

## References to verify during provisioning

- AWS Lightsail: https://aws.amazon.com/lightsail/
- AWS pricing: https://aws.amazon.com/lightsail/pricing/
- Hetzner Cloud: https://www.hetzner.com/cloud
