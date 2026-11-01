# Tasks — SMPP Test Studio

## Current phase: requirements closure (no implementation authorization yet)

### P0 — Design before code
- [x] Record approved scope and decisions in repository.
- [x] Establish `.codex` and docs as durable context.
- [x] Select protocol version for V1: **SMPP 3.4 only**.
- [x] Approve connection defaults: SMPP 3.4, TCP, TRX, one session, 5s connect/10s bind/5s submit-response timeouts, 30s enquire_link interval/10s response timeout, empty system_type/address_range, configurable timers and disable-able automatic enquire_link.
- [x] Approve **Auto + Manual** message encoding selection (D-026): Auto GSM7 when representable, UCS2 for supported other text; manual GSM7/UCS2/Binary and explicit `data_coding`.
- [x] Approve **Multipart Auto + Manual** modes (D-027): default UDH for long messages; manual UDH, SAR TLVs, or No automatic splitting; separate logical/PDU counts.
- [x] Approve **Auto + Manual TON/NPI** selection (D-028) independently for source/destination; no implicit address normalization.
- [x] Approve Auto TON/NPI normal mapping (D-029): digit-only without `+` => `0/0`, explicit `+` => `1/1`, alphanumeric => `5/0`, no address rewrite.
- [x] Approve `registered_delivery=0x00` by default, per-run Request DLR toggle and full-byte Advanced override (D-030).
- [x] Approve Basic DLR ON=`0x01` and Advanced `0x02` / vendor-specific reserved-in-3.4 `0x03` (D-031).
- [x] Confirm `submit_sm` defaults, wire encoding, Basic address validation, multipart and no-split/payload rules (**D-032**).
- [x] Confirm DLR **text-first `id:`**, TLV fallback, state parsing, per-segment correlation and 60s drain (**D-033**).
- [x] Confirm headless outer JSON/YAML+JSON v1 contract and execution exit codes for C++ (**D-034**).
- [ ] Specify nested config/result keys/types, public Go API method names, Advanced TLV list and vendor interoperability fixture plan within the approved D-032–D-034 boundaries.
- [x] Confirm D-035 response selection model.
- [ ] Finalize remaining D-035 delay, status and capacity parameters.
- [ ] Finalize Adaptive Window parameters.
- [ ] Define bounded DLR ID-correlation retention/expiry and local storage/flushing; D-033 already fixes baseline ID and status correlation.
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
