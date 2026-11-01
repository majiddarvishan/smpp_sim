# SMPP Test Studio (`smpp_sim`)

A lightweight, maintainable **SMPP client simulator** for manual testing, bulk/load generation, and automated SMSC integration testing.

> **Project status: requirements and architecture design only. No simulator is implemented yet.**

## Agreed principles

- One shared **Go simulation engine**, used by the CLI, embedded Web UI, headless runner, and importable Go API.
- A **single executable**, with no mandatory external database, broker, monitoring stack, or distributed controller.
- For C++, Python, Java, shell, and other non-Go projects, a **headless JSON/YAML-driven process** returns machine-readable JSON results. The invoking test framework owns all assertions.
- **50,000 `submit_sm` PDUs/s per instance** is the design/benchmark target, **not a verified capability**.
- Outbound MT `submit_sm`, `submit_sm_resp`, concurrent delivery receipts (`deliver_sm`/`deliver_sm_resp`), and configurable response fault injection. **No MO test scenarios.**
- One active Test Run per instance, multiple SMPP sessions; strong preference for simplicity over operational feature growth.

**Confirmed as of 2026-11-01:** D-032 (submit defaults and encoding/fragmentation), D-033 (**DLR text-first `id:`**, TLV fallback, 60s drain), D-034 (headless JSON/YAML/JSON contract and execution exit codes). These are documented decisions, **not implemented features**.

## Documents

| Document | Purpose |
| --- | --- |
| [Product spec](docs/PRODUCT_SPEC.md) | Agreed V1 features and exclusions |
| [Architecture](docs/ARCHITECTURE.md) | Lightweight core, interfaces, run lifecycle |
| [Traffic control](docs/TRAFFIC_CONTROL.md) | Six load patterns; adaptive window; sessions |
| [Protocol, data, faults](docs/PROTOCOL_DATA_FAULTS.md) | PDU/DLR semantics, inputs, fault injection |
| [Headless integration](docs/HEADLESS_INTEGRATION.md) | Go, C++, Python/Java interoperability |
| [Acceptance plan](docs/ACCEPTANCE.md) | Proposed validation requirements |
| [Decision log](docs/DECISIONS.md) | Confirmed decisions |
| [Backlog](docs/BACKLOG.md) | Deferred items and unconfirmed details |

## Agent workflow

Read [`.codex/AGENT_CONTEXT.md`](.codex/AGENT_CONTEXT.md), [`.codex/CURRENT_STATUS.md`](.codex/CURRENT_STATUS.md), and [`.codex/TASKS.md`](.codex/TASKS.md) before touching implementation.

**Documentation-only bootstrap:** this repository is not yet ready to build or run. Examples in docs are illustrative, not approved configuration schemas or measured benchmark results.
