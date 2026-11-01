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
2. **Nested config and API:** D-034+D-038 CONFIRMED the headless invocation, strict v1 outer and minimum nested input/result contracts, `connection.sessions`, finite termination, four-operation Go API and exit codes with no assertions. Remaining implementation details: optional Advanced schema field type spellings, Go types and CLI/Web UI field presentation; do not reopen the approved minimum contract.
3. **Traffic math:** D-037 **CONFIRMED** Adaptive initial/max window (32/1,024), 3s batch deadline versus 5s submit timeout, 1s healthy p95, growth/half-shrink, three severe rounds to zero, 5s single-flight probes, two successes to reopen, 8,192 global pending cap and 10m finite Adaptive run. Still open: precise pacing/config parameters for other five traffic profiles, SMPP congestion-status classification fixtures, bounded late-sequence ledger mechanics and detailed rate-distribution behavior.
4. **Connection lifecycle:** TX rejoin behavior when some TX remain; distinction between temporary RX outage, DLR drain, and all-TX-loss terminal stop; reconnect limits and TRX semantics.
5. **Remaining DLR work:** D-033 fixes **text-first `id:` extraction**, TLV fallback, exact ID matching, status mapping, segment-level eligibility and 60s drain without false Missing for conditional/vendor receipts. Still open: bounded ID-correlation retention/expiry after drain and explicit vendor normalization adapters/interoperability fixtures.
6. **Fault injection:** D-035 confirms the three-way response model and complete DLR override; **D-036 confirms** disabled-by-default Fixed ACK delay, a separate post-selection delay percentage, configurable 60s default delay cap, initial supported status codes (`ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`), default 10,000 pending delayed ACKs and controlled `resource_limit` stop upon exhaustion. Only precise nested configuration naming and implementation mechanics remain.
7. **Data:** CSV schema and invalid-row rules, shuffle implementation for huge files, templates and random strategies, safety restrictions on test destinations.
8. **Storage:** local report files, flush intervals, crash-loss evidence, capture limits and retention; no mandatory DB.
9. **Acceptance:** measurable hardware/conditions for 50k TPS, sustained time, allowed p95/timeout resource use, end-to-end DLR load test workload, cross-platform C++ runner smoke test.
10. **Security and packaging:** secret injection and redaction, UI bind address/access defaults, authorized-target/real-number guardrails, reproducible builds.

## Short next design sequence

- Design advanced TLV allowlist, vendor-specific interoperability fixtures, and bounded DLR correlation-map retention without reopening D-032/D-033.
- Finalize optional **Advanced** schema field types, persistence paths and CLI/Web UI bindings while retaining the confirmed D-034/D-038 minimal API contract.
- Preserve D-037 Adaptive Window tuning; specify nested headless config fields and exact congestion-status classification without reopening the confirmed controller rules.
- Set objective V1 acceptance tests; only then begin coding in small, tested increments.

**Reminder:** no simulator implementation exists at the time of this documentation commit.

## Confirmed design closure — 2026-11-01

**D-032, D-033 and D-034 are CONFIRMED.** The earlier candidate package is no longer pending; its approved details (including the later owner correction requiring **DLR text-first `id:`**) now live in [Protocol](PROTOCOL_DATA_FAULTS.md), [Headless](HEADLESS_INTEGRATION.md) and [Decision Register](DECISIONS.md). D-023 still prohibits implementation until explicitly authorized.

## D-035 design closure

D-035's response-selection model is confirmed in [DECISIONS.md](DECISIONS.md). **D-036 also confirms** Fixed ACK delay defaults, the four initial error statuses, the 60s configurable delay cap, the 10,000-ACK pending limit and controlled `resource_limit` stop. These are not open decisions. Only nested config naming and implementation details remain. No implementation authorized.

## D-037 design closure — 2026-11-01

The owner explicitly **CONFIRMED D-037**. Its numeric controller defaults, batch/submit deadline distinction, severe-degradation threshold, probe safety, bounded outstanding capacity, run deadline and cross-session interactions now live in [DECISIONS.md](DECISIONS.md) and [TRAFFIC_CONTROL.md](TRAFFIC_CONTROL.md#confirmed-adaptive-window-defaults-and-lifecycle-d-037). Do not treat them as proposals. Exact implementation of bounded sequence-ledger/tombstone retention and SMPP congestion-code classification remains engineering design, not a reason to reopen D-037. Implementation is **not authorized**.

## D-038 design closure — 2026-11-01

The owner authorized continuing with the previously offered D-038 package. **D-038 is CONFIRMED** in [DECISIONS.md](DECISIONS.md) and its authoritative full contract is now in [HEADLESS_INTEGRATION.md](HEADLESS_INTEGRATION.md#confirmed-nested-headless-and-go-api-v1-d-038). No legacy `test` group; strict schema v1 with `connection.sessions`, `message`, `traffic`, `completion`, optional `capture`/`faults`/`seed`, finite runs, redacted structured JSON, D-034 execution exit codes and four-operation Go library. C++ invokes the shared binary as subprocess. **No code authorized.**

## Candidate D-039 — Connection recovery, DLR retention and local reports (PENDING OWNER APPROVAL)

**Not confirmed.** One proposed V1 bundle for owner review. This honors the already-confirmed D-013/D-014/D-016/D-017/D-033/D-034/D-036/D-037/D-038; there is **no code authorization**.

### Connection lifecycle and reconnect policy

- For each named TX/RX/TRX session (D-038), reconnect and re-bind inside the same still-running Test Run, with default exponential retry intervals of **1s, 2s, 4s, 8s, 16s, then 30s capped**, configurable and with small jitter to avoid synchronized retries. A permanent bind-credential rejection is recorded as a configuration/authentication diagnostic and is not retried blindly.
- **If one TX/TRX fails**, isolate it and continue sending through other healthy TX/TRX connections according to the selected distribution mode. If it reconnects while another sender remains healthy, rejoin it as a **new bound session generation**, reset the Adaptive controller state for that generation, and assign only *new future messages*. Prior uncertain submits are reported but **never resent**. This preserves D-017 and D-037.
- **If every send-capable session is unavailable at startup**, report startup/connect/bind failure (D-034 exit **3**). If the Test Run had started and **all TX sessions later become unavailable**, terminate immediately with `all_tx_unavailable` (exit **4**) instead of waiting for reconnect. Partial evidence is retained.
- If all RX-capable sessions fail but at least one TX is still healthy, follow D-016's per-run **Continue MT** or **Pause MT** policy. In Pause MT, stop new normal generation/dispatch (no unbounded buffering); keep receiver reconnect attempts running until an RX-capable session returns or a run deadline/stop occurs. A TRX session counts toward both roles. All-TX stop precedence still applies.
- Track per-binding bind attempts/successes/failures, disconnect reason, generation number, downtime, and login redaction. Reconnect never silently changes system_id, credentials, addresses or message-routing policy.

### Bounded DLR correlation and late receipts

- D-033's **text-first DLR `id:`** (TLV ID fallback) is unchanged. Correlation is scoped to a Test Run, across all receiver sessions, with immutable submitted segment identity and explicit handling of duplicate/ambiguous IDs. Never assume submit success means delivered.
- Proposed per-accepted, identifiable segment correlation retention is **120s from accepted `submit_sm_resp`**, configurable and no less than the run's DLR drain default (**60s**). While the test is running, entries expire from the bounded active ID map; after expiration, arriving receipts become **late/unmatched**, never guessed into an unrelated message. Preserve aggregate resolved/missing/expired metrics so capture sampling does not affect totals. Conditional/vendor DLRs (`registered_delivery=0x02/0x03`) do not become guaranteed Missing on expiry.
- Default maximum active correlation entries **1,000,000 per Test Run**, configurable. Do **not** silently evict expected DLRs or treat the absence of a matching ID as delivery failure. If the configured active-map capacity is exhausted, end the run in a controlled way with `resource_limit` and partial evidence (D-034 runtime exit 4). Default DLR request remains OFF, so high-throughput submit-only runs do not need a correlation entry for every message.
- 60s DLR drain after submit completion is still the D-033 default. Stop scheduling new sends in drain; accept DLRs and send ACKs for the allowed drain window. Termination does not imply a global SMSC receipt expiry policy.

### Minimal local evidence and capture

- Default run reporting mode **High Performance**, with accurate atomic aggregate counters and bounded optional sample capture (D-013); Full Detail is an explicit alternative. Limit sample PDU details to **1,000 entries per run** by default, configurable, and raw/full-detail capture to **512 MiB per run** by default, configurable. When capture quota is reached, continue the run but stop **optional raw details**, count `capture_dropped` and set `capture_truncated=true`; do not damage core counters/ID correlation or silently declare complete detail.
- Keep results **locally only**, no mandatory database/broker. CLI and embedded Web UI use a per-run directory beneath a configurable local `runs/` root; Headless uses the explicit D-034/D-038 `--result` file and must not secretly write a second result elsewhere. Raw/sampled artifacts are optional and bounded and referenced from the JSON result when enabled.
- Proposed periodic report/snapshot flush **every 5s**, configurable; finish with atomic replacement where supported and explicit error reporting when the final result cannot be published. After crash/restart, mark the previous run `interrupted` using the available persisted snapshot (D-014), **never auto-resume or resend**. Latest in-memory observations since last successful flush may be lost; record the last successful flush time.
- **No automatic deletion of run artifacts in V1**, to avoid accidental loss of user's test evidence. Provide explicit manual cleanup; sensitive fields/credentials must always be redacted, filenames sanitized and local permissions conservative. On disk-full/result-write failure, report a runtime/storage diagnostic and preserve remaining evidence where feasible; do not silently continue promising a complete report.

### Checks before implementation

- Verify startup all-TX failure versus runtime all-TX loss, per-TX reconnect generations, new-only traffic after rebind, RX loss Continue/Pause and TRX role overlap; all under bounded run duration.
- Verify receipt arrival before/after 120s TTL, text/TLV ID conflict, duplicates/ambiguity, DLR drain, DLR OFF high-throughput behavior and configured 1,000,000 correlation-cap stop; no fabricated DLR failure.
- Verify 5s local snapshot, controlled crash -> `interrupted` without replay, final JSON write attempt/exit behavior, 512 MiB optional capture truncation without changing aggregate counts, no background cleanup or mandatory external storage. These are **planned** acceptance tests only.

**Owner review:** approve/revise/reject D-039 as a single package. Upon approval, add a CONFIRMED register entry and synchronize ARCHITECTURE, PROTOCOL_DATA_FAULTS, PRODUCT_SPEC, ACCEPTANCE, BACKLOG, and .codex. Until then this is only a proposal.
