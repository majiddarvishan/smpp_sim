# Tasks — SMPP Test Studio

## Current phase: requirements closure (no implementation authorization yet)

### P0 — Design before code
- [x] Record approved scope and decisions in repository.
- [x] Establish `.codex` and docs as durable context.
- [ ] Finalize SMPP session fields/defaults (TCP bind modes; credentials; TON/NPI; timers; enquire_link; SMPP version).
- [ ] Finalize versioned config and result schema for CLI/Web UI/headless C++/Go integration, **without built-in assertions**.
- [ ] Finalize deterministic fault-response policy and all Adaptive Window parameters.
- [ ] Define DLR correlation, storage and error classification details.
- [ ] Approve measurable V1 acceptance and benchmark workload.

### P1 — Implementation *after* user requests it
- [ ] Minimal Go SMPP client core, PDU and session lifecycle plus unit tests.
- [ ] Single-message send + DLR ACK and correlation.
- [ ] Shared API + headless runner (JSON/YAML input, JSON results); C++ cross-language smoke.
- [ ] Streaming input/data generator; six load patterns and independent Adaptive Windows.
- [ ] Connection distribution, fault injection, capture/reporting.
- [ ] CLI + embedded Web UI, one active run; full acceptance suite.

### Deferred
See `docs/BACKLOG.md`: TLS, `query_sm`, `cancel_sm`, `replace_sm`. No MO scenarios.

No tasks may be marked complete based on design intent alone; test execution results are not yet available.
