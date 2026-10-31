# Personal AI Operating System

A self-hosted personal assistant built around [OpenClaw](https://docs.openclaw.ai/) as the gateway/orchestration runtime, with custom integrations (GitHub, AWS, herdr), workflows, and observability.

## Status

Early design phase. Architectural decisions are recorded as ADRs in [`docs/decisions/`](docs/decisions/):

- [ADR-001: OpenClaw over ChatGPT Desktop](docs/decisions/001-openclaw-vs-chatgpt-desktop.md)
- [ADR-002: Cloud-first always-on execution over a dedicated Mac mini](docs/decisions/002-remote-vps-vs-mac-mini.md)
- [ADR-003: AWS Lightsail for MVP hosting, with Hetzner portability](docs/decisions/003-aws-lightsail-vs-hetzner.md)

Runbooks (how to deploy and operate) live in [`docs/runbooks/`](docs/runbooks/):

- [Runbook 001: Lightsail server setup and first SSH connection](docs/runbooks/001-lightsail-server-setup.md)

## Architecture rule

Do not fork or copy OpenClaw into this repository. Keep it as an upstream dependency and version its configuration under `config/`. Keep herdr independently maintained and integrate through its CLI or a stable API.

## Layout

| Path | Purpose |
|---|---|
| `docs/` | Architecture, decisions (ADRs), runbooks, evaluations, experiments |
| `config/` | Versioned OpenClaw and agent configuration |
| `src/` | Integrations, workflows, tools, observability, security |
| `infra/` | AWS, Docker, and bootstrap scripts |
| `tests/` | Unit, integration, and eval tests |
| `examples/` | Usage examples |

## Secrets

Never commit secrets. Copy `.env.example` to `.env` locally.
