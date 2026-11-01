# Backlog, Exclusions and Open Decisions

Status: 2026-11-01. Do not treat candidate defaults as approved.

## Deferred beyond V1

- SMPP protocol versions other than **3.4** (only if a real need arises).
- **TLS** for SMPP connections.
- **`query_sm`**, **`cancel_sm`**, **`replace_sm`**.
- Potential features beyond this list require explicit scope approval; avoid building a platform.

## Explicitly out of scope

- MO traffic generation, MO receipt test scenarios, MO injection tools.
- Built-in assertion rules/DSL or test-management service.
- Automatic resume/checkpoint replay after process restart/crash.
- Required external broker/database/distributed worker cluster.
- Native C++ SDK/FFI (use headless subprocess JSON results); separate orchestration server.
- SMSC-side `deliver_sm` generator/server implementation (this is a client simulator).

## Unsettled details to finalize before implementation

1. **SMPP parameters:** SMPP 3.4 only; connection defaults (D-024), optional automatic enquire_link (D-025), Auto + Manual encoding (D-026), multipart mode selection (D-027), and Auto + Manual TON/NPI **selection policy** (D-028) are approved. D-032/D-033 now settle Basic malformed addresses, bind version, baseline optional fields, GSM7/UCS2/binary wire defaults, non-BMP handling, UDH/SAR, no-split/payload and per-segment DLR matching. Still open: unbind timeout, Advanced TLV catalog, manual DCS/UCS2 mismatch UX, and vendor interoperability fixtures.
2. **Nested config and API:** D-034 fixes headless command, versioned input/output outer groups, finite execution and exit codes with **no built-in assertions**. Still open: exact nested key names/types/default serialization, Go API method signatures, and CLI/Web UI field presentation.
3. **Traffic math:** precise pacing clock/limits, Adaptive increase/decrease/timeout thresholds, max/initial window defaults, zero-window transition and probe interval/success count, late response bookkeeping, per-session vs global controls.
4. **Connection lifecycle:** TX rejoin behavior when some TX remain; distinction between temporary RX outage, DLR drain, and all-TX-loss terminal stop; reconnect limits and TRX semantics.
5. **Remaining DLR work:** D-033 fixes **text-first `id:` extraction**, TLV fallback, exact ID matching, status mapping, segment-level eligibility and 60s drain without false Missing for conditional/vendor receipts. Still open: bounded ID-correlation retention/expiry after drain and explicit vendor normalization adapters/interoperability fixtures.
6. **Fault injection:** D-035 confirms the three-way response model and complete DLR override; **D-036 confirms** disabled-by-default Fixed ACK delay, a separate post-selection delay percentage, configurable 60s default delay cap, initial supported status codes (`ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`), default 10,000 pending delayed ACKs and controlled `resource_limit` stop upon exhaustion. Only precise nested configuration naming and implementation mechanics remain.
7. **Data:** CSV schema and invalid-row rules, shuffle implementation for huge files, templates and random strategies, safety restrictions on test destinations.
8. **Storage:** local report files, flush intervals, crash-loss evidence, capture limits and retention; no mandatory DB.
9. **Acceptance:** measurable hardware/conditions for 50k TPS, sustained time, allowed p95/timeout resource use, end-to-end DLR load test workload, cross-platform C++ runner smoke test.
10. **Security and packaging:** secret injection and redaction, UI bind address/access defaults, authorized-target/real-number guardrails, reproducible builds.

## Short next design sequence

- Design advanced TLV allowlist, vendor-specific interoperability fixtures, and bounded DLR correlation-map retention without reopening D-032/D-033.
- Finalize exact **nested** JSON/YAML config/result types and Go API signatures within the already confirmed D-034 outer contract.
- Freeze Adaptive Window tuning and the remaining nested config schema (D-035/D-036 fault-selection and timing rules are now confirmed).
- Set objective V1 acceptance tests; only then begin coding in small, tested increments.

**Reminder:** no simulator implementation exists at the time of this documentation commit.

## Confirmed design closure — 2026-11-01

**D-032, D-033 and D-034 are CONFIRMED.** The earlier candidate package is no longer pending; its approved details (including the later owner correction requiring **DLR text-first `id:`**) now live in [Protocol](PROTOCOL_DATA_FAULTS.md), [Headless](HEADLESS_INTEGRATION.md) and [Decision Register](DECISIONS.md). D-023 still prohibits implementation until explicitly authorized.

## D-035 design closure

D-035's response-selection model is confirmed in [DECISIONS.md](DECISIONS.md). **D-036 also confirms** Fixed ACK delay defaults, the four initial error statuses, the 60s configurable delay cap, the 10,000-ACK pending limit and controlled `resource_limit` stop. These are not open decisions. Only nested config naming and implementation details remain. No implementation authorized.
