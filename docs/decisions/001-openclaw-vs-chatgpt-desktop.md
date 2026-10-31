# ADR-001: OpenClaw over ChatGPT Desktop for the personal assistant runtime

- **Status:** Accepted for MVP; revisit after operational testing
- **Date:** 2026-10-31
- **Decision owners:** Herd project
- **Scope:** Personal assistant orchestration, not the choice of coding model or coding-agent harness

## Context

We want a personal assistant reachable from a phone, able to execute cloud-accessible work while the user's Mac is off, coordinate project and coding workflows, and integrate with custom tools including herdr. The system should allow model/provider changes and expose enough execution history to debug failures. We also want to publish an engineering case study about its design.

## Decision

Use **OpenClaw as the self-hosted assistant gateway and orchestration runtime**. Use ChatGPT separately as a complementary interactive assistant, not as the persistent execution backbone. Keep **herdr** as an experimental coding-execution integration pending comparison with native OpenClaw delegation.

## Why

1. A self-hosted gateway can run continuously on infrastructure we control.
2. We can implement and deploy custom tool adapters, task policies, notifications, and audit trails.
3. Model/provider choice can be changed without replacing the entire assistant architecture, subject to provider compatibility and terms.
4. Remote messaging and a gateway/node topology fit the requirement to work when the Mac is unavailable.
5. Running the system ourselves creates useful learning opportunities in security, observability, integration, and reliability.

## Alternatives considered

- **ChatGPT Desktop / ChatGPT Work:** Strong interactive user experience and integrated capabilities, but not selected as the self-managed, always-on gateway for arbitrary custom workers and our own infrastructure. Product capabilities evolve; this is a fit-for-purpose decision, not a claim that ChatGPT lacks all automation or integrations.
- **Custom agent runtime from scratch:** Maximum control and learning, but slower to useful operation. Build a small educational agent loop separately.
- **Other orchestration platforms:** Deferred until a concrete gap appears in the MVP.

## Consequences and risks

- We operate, update, secure, and troubleshoot the gateway ourselves.
- Model API or eligible subscription access may incur usage limits or charges; verify current provider terms before choosing an authentication path.
- A compromised agent with broad tool access can cause serious damage. Use least privilege, isolated credentials, and explicit approval for destructive or expensive actions.
- Tool support and integration quality must be verified through tests, not assumed.

## MVP acceptance criteria

- Send an authenticated message from a phone and receive a response while the Mac is powered off.
- Read a permitted GitHub repository and produce a status report with supporting references.
- Record task ID, timestamps, outcome, tool errors, and approval decisions.
- Demonstrate a provider-switching path without rewriting the business logic.

## Revisit if

- Operational overhead consistently exceeds the benefits.
- An alternative offers equivalent extensibility, unattended execution, security, and portability with substantially less complexity.
- OpenClaw cannot reliably support required tools or model providers.

## References to verify during implementation

- OpenClaw official documentation: https://docs.openclaw.ai/
- ChatGPT official product documentation: https://help.openai.com/
