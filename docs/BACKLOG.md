# Backlog, Exclusions and Open Decisions

Status: 2026-11-01. D-001–D-040 are owner-confirmed. Engineering implementation details below are not additional owner-level design decisions.

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

## Scoped implementation and interoperability details (not P0 owner decisions)

1. **SMPP parameters:** SMPP 3.4 only; connection defaults (D-024), optional automatic enquire_link (D-025), Auto + Manual encoding (D-026), multipart mode selection (D-027), and Auto + Manual TON/NPI **selection policy** (D-028) are approved. D-032/D-033 now settle Basic malformed addresses, bind version, baseline optional fields, GSM7/UCS2/binary wire defaults, non-BMP handling, UDH/SAR, no-split/payload and per-segment DLR matching. Still open: unbind timeout, Advanced TLV catalog, manual DCS/UCS2 mismatch UX, and vendor interoperability fixtures.
2. **Nested config and API:** D-034+D-038 CONFIRMED the headless invocation, strict v1 outer and minimum nested input/result contracts, `connection.sessions`, finite termination, four-operation Go API and exit codes with no assertions. Remaining implementation details: optional Advanced schema field type spellings, Go types and CLI/Web UI field presentation; do not reopen the approved minimum contract.
3. **Traffic math:** D-037 **CONFIRMED** Adaptive initial/max window (32/1,024), 3s batch deadline versus 5s submit timeout, 1s healthy p95, growth/half-shrink, three severe rounds to zero, 5s single-flight probes, two successes to reopen, 8,192 global pending cap and 10m finite Adaptive run. Still open: precise pacing/config parameters for other five traffic profiles, SMPP congestion-status classification fixtures, bounded late-sequence ledger mechanics and detailed rate-distribution behavior.
4. **Connection lifecycle:** D-039 confirms reconnect backoff, returned-TX new bind generations, RX-only Continue/Pause with in-process reconnect, startup versus runtime all-TX loss and distinct exit codes. Remaining: detailed bind-generation fencing and cancellation scheduling, not new product decisions.
5. **Remaining DLR work:** D-033 and D-039 confirm text-first ID, segment-level 60s drain, configurable 120s active-map TTL, 1,000,000 cap and `resource_limit` stop. Remaining: explicit vendor normalization adapters/interoperability fixtures and bounded matching implementation.
6. **Fault injection:** D-035 confirms the three-way response model and complete DLR override; **D-036 confirms** disabled-by-default Fixed ACK delay, a separate post-selection delay percentage, configurable 60s default delay cap, initial supported status codes (`ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`), default 10,000 pending delayed ACKs and controlled `resource_limit` stop upon exhaustion. Only precise nested configuration naming and implementation mechanics remain.
7. **Data:** CSV schema and invalid-row rules, shuffle implementation for huge files, templates and random strategies, safety restrictions on test destinations.
8. **Storage:** D-039 fixes local-only reports, High Performance default, 1,000 samples, 512 MiB optional raw capture, 5s snapshot flush, crash-interrupted behavior and no automatic deletion. Remaining: atomic file-format mechanics, permission checks and observable disk-error handling.
9. **Acceptance:** D-040 establishes a reproducible five-minute 50k physical-submit PDU/s reference profile with two runs, <8 GiB RSS, verified peer capacity, plus a separate 5k submit PDU/s three-minute DLR workload and C++ smoke. Implementation must produce measured evidence; no benchmark was run at design freeze.
10. **Security and packaging:** D-040 requires finite tests, explicit authorized destinations and probes, localhost-only embedded UI default, environment-secret redaction, conservative local reports, Linux/Windows amd64 release smoke. Detailed platform packaging and validation tests belong to implementation.

## Next engineering sequence (after separate implementation authorization)

- Design advanced TLV allowlist and vendor-specific interoperability fixtures; D-039 already fixes DLR mapping retention/cap.
- Finalize optional **Advanced** schema field types, persistence paths and CLI/Web UI bindings while retaining the confirmed D-034/D-038 minimal API contract.
- Preserve D-037 Adaptive Window tuning; specify nested headless config fields and exact congestion-status classification without reopening the confirmed controller rules.
- Implement the D-040 acceptance suite and reference benchmark harness in small, tested increments; report results only after execution.

**Reminder:** V1 design approval alone does not authorize implementation and does not establish benchmark PASS.

## Confirmed design closure — 2026-11-01

**D-032, D-033 and D-034 are CONFIRMED.** The earlier candidate package is no longer pending; its approved details (including the later owner correction requiring **DLR text-first `id:`**) now live in [Protocol](PROTOCOL_DATA_FAULTS.md), [Headless](HEADLESS_INTEGRATION.md) and [Decision Register](DECISIONS.md). D-023 still prohibits implementation until explicitly authorized.

## D-035 design closure

D-035's response-selection model is confirmed in [DECISIONS.md](DECISIONS.md). **D-036 also confirms** Fixed ACK delay defaults, the four initial error statuses, the 60s configurable delay cap, the 10,000-ACK pending limit and controlled `resource_limit` stop. These are not open decisions. Only nested config naming and implementation details remain. No implementation authorized.

## D-037 design closure — 2026-11-01

The owner explicitly **CONFIRMED D-037**. Its numeric controller defaults, batch/submit deadline distinction, severe-degradation threshold, probe safety, bounded outstanding capacity, run deadline and cross-session interactions now live in [DECISIONS.md](DECISIONS.md) and [TRAFFIC_CONTROL.md](TRAFFIC_CONTROL.md#confirmed-adaptive-window-defaults-and-lifecycle-d-037). Do not treat them as proposals. Exact implementation of bounded sequence-ledger/tombstone retention and SMPP congestion-code classification remains engineering design, not a reason to reopen D-037. Implementation is **not authorized**.

## D-038 design closure — 2026-11-01

The owner authorized continuing with the previously offered D-038 package. **D-038 is CONFIRMED** in [DECISIONS.md](DECISIONS.md) and its authoritative full contract is now in [HEADLESS_INTEGRATION.md](HEADLESS_INTEGRATION.md#confirmed-nested-headless-and-go-api-v1-d-038). No legacy `test` group; strict schema v1 with `connection.sessions`, `message`, `traffic`, `completion`, optional `capture`/`faults`/`seed`, finite runs, redacted structured JSON, D-034 execution exit codes and four-operation Go library. C++ invokes the shared binary as subprocess. **No code authorized.**

## Confirmed D-039 design closure — 2026-11-01

D-039 is **CONFIRMED** by the owner. Its complete approved policy is in [DECISIONS.md](DECISIONS.md), [ARCHITECTURE.md](ARCHITECTURE.md#confirmed-connection-correlation-and-local-evidence-d-039) and [PROTOCOL_DATA_FAULTS.md](PROTOCOL_DATA_FAULTS.md#confirmed-dlr-correlation-retention-d-039). Confirmed defaults include 1/2/4/8/16/30s-capped reconnect backoff; new bind generations for returned TXs; startup exit 3 versus runtime all-TX-loss exit 4; 120s DLR mapping retention with 1,000,000 active-entry limit; default High Performance, 1,000 samples, 512 MiB capture ceiling, 5s snapshots; local run reports, no automatic deletion. These are **not pending**. No implementation authorization.

## Confirmed D-040 design acceptance — 2026-11-01

D-040 is **CONFIRMED** by the owner, recorded in [DECISIONS.md](DECISIONS.md) and [ACCEPTANCE.md](ACCEPTANCE.md#confirmed-v1-acceptance-and-benchmark-gate-d-040). All P0 product behavior has a documented default, invariant or scoped exclusion in decisions D-001–D-040. The 50,000 physical submit PDU/s for five minutes and 5,000 submit PDU/s with DLRs for three minutes are **future measured acceptance objectives**, not completed tests. A documented consistency audit is mandatory before declaring V1 design FROZEN. D-023 still prohibits implementation until separately authorized.
