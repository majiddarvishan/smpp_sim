# Tasks — SMPP Test Studio

## Current phase: requirements closure (no implementation authorization yet)

### P0 — Design before code
- [x] Record approved scope and decisions in repository.
- [x] Establish `.codex` and docs as durable context.
- [x] Select protocol version for V1: **SMPP 3.4 only**.
- [x] Approve connection defaults: SMPP 3.4, TCP, TRX, one session, 5s connect/10s bind/5s submit-response timeouts, 30s enquire_link interval/10s response timeout, empty system_type/address_range, configurable timers and disable-able automatic enquire_link.
- [x] Approve **Auto + Manual** message encoding selection (D-026): Auto GSM7 when representable, UCS2 for supported other text; manual GSM7/UCS2/Binary and explicit `data_coding`.
- [ ] Finalize remaining `submit_sm` and DLR parameters: TON/NPI, exact GSM7 packing/DCS interoperability, unsupported Unicode, multipart UDH/SAR, optional PDU fields, message ID normalization.
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
