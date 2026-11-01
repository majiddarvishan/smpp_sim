# Product Requirements — SMPP Test Studio

Status: **requirements agreed; implementation not started**. Updated 2026-11-01.

## Purpose and scope

A lightweight SMPP **client** simulator that can send individual and bulk MT messages, inspect submit responses, receive delivery receipts, inject controlled faults in `deliver_sm_resp`, and measure SMSC behavior under high load. The same engine must support humans (CLI/Web UI) and automation (Go library and language-neutral headless runner).

- Throughput design target: **50,000 `submit_sm` PDUs/sec in one simulator instance**, subject to measurement on defined hardware and SMSC test environment.
- Distinguish logical SMS messages, multipart segments, submitted PDUs, accepted `submit_sm_resp`, completed DLR, and Actual vs Target TPS.
- No MO generation, reception test scenario, or MO injector in product scope.
- No need to implement an SMS gateway, billing, production delivery system, generic test orchestrator, or full protocol server.

## Interface modes

1. CLI: manual one-message and bulk use.
2. Embedded Web UI: create/control one active run, monitor connections, chart metrics, inspect bounded PDU samples, retrieve reports.
3. Headless runner: load JSON/YAML config, run without interactive CLI/UI or web listener, emit JSON result and meaningful process status for integration tests in C++, Python, Java, etc.
4. Importable Go API: direct programmatic test setup, `context` timeout/cancellation, send/wait/stop, structured results. CLI/UI/headless must reuse this engine.

## SMPP functions in V1

- **SMPP v3.4 only** for V1. Plain TCP connections with configurable host/port, system ID/password, `bind_transmitter`, `bind_receiver`, and `bind_transceiver` (TX/RX/TRX).
- Multiple concurrent sessions, lifecycle visibility, optional automatic `enquire_link`, bind/rebind and timeouts; TX and DLR RX can be separate or TRX.

### Approved connection defaults

| Setting | Default |
| --- | --- |
| SMPP version | 3.4 |
| Transport | TCP (TLS deferred) |
| Bind mode | TRX |
| Connection count | 1 |
| Connect timeout | 5s |
| Bind timeout | 10s |
| Submit response timeout | 5s |
| Enquire Link interval | 30s |
| Enquire Link response timeout | 10s |
| System Type (`system_type`) | Empty |
| Address Range (`address_range`) | Empty |

All timeout values are configurable. Automatic `enquire_link` can be **disabled** (while inbound protocol handling still works). Host/port/credentials must be configured, not assumed. The 5-second per-submit response timeout differs from the Adaptive Window **batch deadline**, for which defaults are not yet approved. Config field names and disabled-value representation are not finalized.
- `submit_sm` creation with primary SMPP fields (TON/NPI, `esm_class`, `registered_delivery`, `data_coding`, validity/scheduling fields where applicable, and configurable TLVs as supported).
- Fixed and templated message content; GSM7/UCS2/binary cases, short and multipart UDH/SAR test cases.
- **Approved Multipart UX:** default **Auto**, which fragments long messages using **UDH**. Advanced users can choose **UDH**, **SAR TLVs** or **No automatic splitting**. For manual UDH/SAR selection use that method for long texts; short-message handling must not add unnecessary multipart headers. Exact length thresholds, header/reference construction, and no-split oversize behavior are still open. Never silently truncate or re-encode the data.
- **Multipart reporting:** count logical messages separately from `submit_sm` PDUs/segments; per-segment response outcomes are observable for both tests and load benchmarks. All invocation modes share this behavior.
- **Approved encoding UX:** Basic view defaults to **Auto** (GSM 03.38 default/extension alphabet when every character is representable; otherwise UCS2 for supported characters, including Persian). Advanced view provides manual GSM7, UCS2, or binary plus explicit `data_coding` configuration. CLI, Web UI and headless must use identical encoding-selection rules. Segment/PDU accounting must use encoded lengths, not unencoded character counts. Exact SMPP wire mapping and unsupported Unicode handling remain open.
- Concurrent `submit_sm_resp` matching by `sequence_number`, timeouts, negative statuses, and late responses. **Never blindly retry an uncertain submit.**
- DLR receipt via `deliver_sm`, appropriate `deliver_sm_resp`, classification, cross-session message ID correlation, duplicate/unmatched/missing/late report metrics.
- Per-run completion choice: **Submit Completion** or **Submit + DLR Completion** with configurable DLR drain deadline.

## Traffic profiles — all six in V1

1. Constant TPS.
2. Ramp-up/Ramp-down.
3. Step Load.
4. Burst/Spike.
5. Unlimited (limited by available resources, in-flight bounds, and safety limits).
6. Adaptive Window: send a batch up to current window, wait for associated submit responses until a deadline; shrink on congestion/missing responses; grow gradually on successful rounds up to max; allow zero-window Critical Pause and controlled automatic recovery probes.

Per-session Adaptive Window is independent. Run-wide traffic allocation is configurable: Equal, Adaptive, or Weighted. A global limiter governs capped profiles. Some mode combinations and priority rules are still to be specified explicitly.

## Data generation

- Fixed, sequential-range, random-prefix, list, CSV, and weighted destination/source strategies; streaming/lazy generation rather than preloading millions of records.
- Content: fixed/random/template; variable size; encoding and multipart configurations.
- For finite datasets: choose **Stop at End** or **Loop**, and **Sequential** or **Shuffle** ordering; optionally deterministic seed.
- Shuffle of large datasets may require a bounded disk-backed index rather than full in-memory shuffling.
- Reaching dataset end in Stop mode is a distinct termination reason, not an SMPP failure.
- Test destinations must be explicitly authorized; prevent accidental real-network bulk sends.

## Metrics and capture

- Accurate aggregate counters and latency distributions even if detailed PDU capture is sampled/off.
- Per-session and total TPS, accepted/rejected/timeouts, in-flight, window, bind state, RX outage gaps, DLR states, error-code breakdown, fault-injection outcomes.
- Per-run capture mode: **Full Detail** or **High Performance**, with optional bounded **Sample Capture** and selected-error details.
- Bounded memory and queues; storage capture must not block the hot path. Report capture overflow explicitly.
- Persist available run metadata/counters/reports locally with JSON/CSV exports; exact file layout/retention policy pending.

## Run failure/lifecycle

- When one TX session disconnects, continue with other healthy TX sessions; exclude failed TX from normal traffic routing.
- When **all TX sessions are unavailable, stop the run**; never pause the clock waiting for all TX to return and never auto-resume that run.
- When all DLR-capable RX connections disappear but some TX remain: per-run policy **Continue MT** or **Pause MT**. Retry RX binding in the same live process and resume paused sending when a suitable RX bind returns.
- If process crashes/restarts, mark formerly active runs **Interrupted**, keep persisted evidence, and never auto-resume. Some counters after last successful flush may be unavailable.
- One active run per simulator instance; user starts a fresh run for retesting.

## Not V1

TLS, `query_sm`, `cancel_sm`, `replace_sm` are **backlog**. MO scenarios are **excluded** rather than deferred by default. No required orchestration services, native C++ SDK, server-side assertions, or automatic restart checkpoints.
