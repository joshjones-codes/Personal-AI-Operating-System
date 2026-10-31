# Architecture overview

```text
personal-ai-os
├── OpenClaw             Gateway, messaging, agent sessions
├── Custom integrations  GitHub, AWS, herdr
├── Workflows            Task execution, approvals, notifications
├── Infrastructure       AWS initially, portable to Hetzner
└── Observability        Logs, traces, metrics, evaluations
```

See [`../decisions/`](../decisions/) for the reasoning behind each choice.
