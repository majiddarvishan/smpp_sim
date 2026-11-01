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
- Finalize exact **nested** JSON/YAML config/result types and Go API signatures within the already confirmed D-034 outer contract.
- Preserve D-037 Adaptive Window tuning; specify nested headless config fields and exact congestion-status classification without reopening the confirmed controller rules.
- Set objective V1 acceptance tests; only then begin coding in small, tested increments.

**Reminder:** no simulator implementation exists at the time of this documentation commit.

## Confirmed design closure — 2026-11-01

**D-032, D-033 and D-034 are CONFIRMED.** The earlier candidate package is no longer pending; its approved details (including the later owner correction requiring **DLR text-first `id:`**) now live in [Protocol](PROTOCOL_DATA_FAULTS.md), [Headless](HEADLESS_INTEGRATION.md) and [Decision Register](DECISIONS.md). D-023 still prohibits implementation until explicitly authorized.

## D-035 design closure

D-035's response-selection model is confirmed in [DECISIONS.md](DECISIONS.md). **D-036 also confirms** Fixed ACK delay defaults, the four initial error statuses, the 60s configurable delay cap, the 10,000-ACK pending limit and controlled `resource_limit` stop. These are not open decisions. Only nested config naming and implementation details remain. No implementation authorized.

## D-037 design closure — 2026-11-01

The owner explicitly **CONFIRMED D-037**. Its numeric controller defaults, batch/submit deadline distinction, severe-degradation threshold, probe safety, bounded outstanding capacity, run deadline and cross-session interactions now live in [DECISIONS.md](DECISIONS.md) and [TRAFFIC_CONTROL.md](TRAFFIC_CONTROL.md#confirmed-adaptive-window-defaults-and-lifecycle-d-037). Do not treat them as proposals. Exact implementation of bounded sequence-ledger/tombstone retention and SMPP congestion-code classification remains engineering design, not a reason to reopen D-037. Implementation is **not authorized**.

## Candidate D-038 — Minimal Headless/Go API V1 contract (PENDING OWNER APPROVAL)

**Not confirmed.** Design-only proposal for the next review. Preserve D-018/D-019/D-024–D-036 and the D-037 Adaptive controller. No new service, assertion DSL, native C++ binding or second executable.

### Minimal input (JSON and YAML same schema)

- Exactly one top-level `schema_version: 1` and these groups: `connection`, `message`, `traffic`, `completion`, optional `capture`, `faults`, `seed`. No legacy `test` top-level alias. Unknown fields, duplicate YAML mapping keys, mismatched types and unsupported values fail validation (headless exit 2).
- `connection.sessions` is a **nonempty list** of named SMPP bindings; one TRX entry is the usual Basic default. Each session specifies `id`, `host`, `port`, `system_id`, `password_env`, `bind_mode` (tx/rx/trx); optional `weight` (positive), optional per-session timeouts and enquire_link options inherit D-024/D-025. Do not accept cleartext passwords in committed configs or serialized results. Preserve separate RX/TX and multiple sessions without new connection schema versions.
- `message` supports a fixed source/destination/text input for small integration tests and explicit alternatives for binary and dataset-driven work. Initial fixed-message keys: `source`, `destination`, `text`, `encoding` (default auto), `multipart` (default auto), `request_dlr` (default false), optional `registered_delivery` raw override with priority per D-030/D-031, optional `data_coding`. Advanced source/destination TON/NPI overrides use individual fields; never modify raw addresses. Binary octets must be supplied as explicitly base64-encoded bytes rather than a lossy text string. Dataset/CSV options remain nested in `message`, not a new top-level service.
- `traffic` includes `mode` (six approved profiles), finite `count`, `duration`, or finite Stop-at-End input, applicable mode-specific settings (e.g. `tps`, `adaptive`), and `distribution` (equal/adaptive/weighted). Refuse a run with no finite termination. D-037 Adaptive defaults include a finite default maximum wall-clock duration of 10m; do not silently turn on Unlimited without a cap.
- `completion` includes `mode` (submit or submit_and_dlr) and `dlr_timeout` (default 60s per D-033). `capture` contains bounded mode/sample settings; `faults` contains D-035/D-036 generic and DLR-specific ACK policies; `seed` is an integer, effective seed surfaced in redacted results for reproducible generation. All interfaces resolve these defaults through one core config model.
- Relative input dataset paths resolve against the directory of the config file, not the caller's working directory. `--result` is an explicit path controlled by the caller, outside the config schema.

### Minimal result and process contract

- Every final JSON report has `schema_version`, `run_id`, `run_status`, `termination_reason`, `started_at`, `finished_at`, redacted `effective_config`, `counters`, `per_session`, `dlr`, `diagnostics` (D-034 already fixes these outer keys). Use UTC RFC3339 timestamps and integer counters; latency and durations have explicit millisecond units. Report partial counters when a run stops/errors; logs go to stderr, not the JSON result.
- Proposed `run_status`: `completed` (requested finite work finished), `stopped` (manual/safety/all-TX loss), `cancelled`, `error`, and persisted historical `interrupted` after crash. `termination_reason` distinguishes `completed`, `dataset_exhausted`, `all_tx_unavailable`, `resource_limit`, `run_deadline`, `cancelled`, `startup_failure`, `internal_error`. Avoid interpreting SMSC negative responses or missing DLRs as simulator execution/assertion failure by themselves.
- Exit codes stay **exactly** D-034: 0 normal completed even if SMSC rejects messages/missing DLR, 2 config, 3 startup/bind, 4 runtime failure/all-TX/resource_limit, 5 cancelled/deadline, 6 internal/result-file error. Attempt atomic JSON result publication even on startup failure; if result cannot be written, exit 6 and explain on stderr. No non-JSON status text on stdout.
- Counters distinguish **logical messages** and **physical PDUs**, response success/rejection/timeout/uncertainty, segment outcomes, actual recovery probes, DLR delivered/failed/missing/duplicate/unmatched/ambiguous/late, effective DLR text-first message-ID conflict diagnostics, and ACK faults selected versus sent. Explicitly label non-additive event counters so C++ tests do not double-count outcomes. Details are bounded; aggregate metrics stay accurate without capture.

### Go library and C++ contract

- One importable Go package with a small lifecycle: `New(Config) (Runner,error)`, `Runner.Run(context.Context) (Result,error)`, `Runner.Stop()`, `Runner.Close()` (idempotent). The CLI and Web UI call the same engine. Result + error reflects execution transport/config problems; no built-in test assertions.
- C++ integration directly runs `smpp-studio run --headless --config ... --result ...` using an argument array, temporary config/result directory and process deadline, reads JSON and asserts with Google Test/Catch2. No C++ SDK, shared object, service endpoint, or interactive script.
- Explicitly reject plaintext secret output; credentials supplied through environment or a separately approved secure injection method only. Support cancellation cleanup and no automatic replay of ambiguous submits.

**Owner review needed:** approve/revise D-038 as one package; only upon approval append a CONFIRMED register entry and move details into `docs/HEADLESS_INTEGRATION.md`, `docs/PRODUCT_SPEC.md`, `docs/ACCEPTANCE.md` and `.codex`. Do not invent implementation PASS claims.
