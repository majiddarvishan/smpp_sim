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
3. Headless runner: use the confirmed D-034/D-038 single executable and strict schema-v1 JSON/YAML with `connection.sessions` (named TX/RX/TRX), fixed/binary/dataset input inside `message`, finite `traffic`, `completion`, optional bounded `capture`/`faults` and seed. Emit redacted structured JSON with D-034 execution exit codes for C++, Python, Java, etc., without CLI prompts/UI or assertions.
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
- **Approved TON/NPI UX:** Basic defaults to **Auto**, selecting source and destination TON/NPI independently based on recognizable address form. Alphanumeric addresses may use `TON=5, NPI=0` and explicitly indicated international addresses `TON=1, NPI=1`; digit-only addresses without a leading `+` use `TON=0, NPI=0` (D-029). Advanced allows explicit individual overrides of source TON, source NPI, destination TON and destination NPI. The transmitted **address string is never silently rewritten, stripped, or internationalized** by TON/NPI detection; explicit normalization would be a separate user-selected function, not part of this decision. All interfaces share one rule.
- **Confirmed Auto TON/NPI defaults (D-029):** for source and destination independently:
  
  | Input address | TON | NPI |
  | --- | --- | --- |
  | `MyBrand` (alphanumeric) | 5 | 0 |
  | `+989121234567` (explicit international) | 1 | 1 |
  | `989121234567` (digits, no `+`) | 0 | 0 |
  | `1000` (short code, no `+`) | 0 | 0 |

  Advanced manual TON/NPI always overrides these defaults. The **literal configured address is never changed**. Basic malformed-address validation is settled in D-032; explicit Advanced noncanonical ASCII is separately allowed.
- Fixed and templated message content; GSM7/UCS2/binary cases, short and multipart UDH/SAR test cases.
- **Approved Multipart UX:** default **Auto**, which fragments long messages using **UDH**. Advanced users can choose **UDH**, **SAR TLVs** or **No automatic splitting**. For manual UDH/SAR selection use that method for long texts; short-message handling must not add unnecessary multipart headers. D-032 fixes segmentation budgets, reference fields and no-split behavior; vendor interoperability still requires tests. Never silently truncate or re-encode the data.
- **Multipart reporting:** count logical messages separately from `submit_sm` PDUs/segments; per-segment response outcomes are observable for both tests and load benchmarks. All invocation modes share this behavior.
- **Approved encoding UX:** Basic view defaults to **Auto** (GSM 03.38 default/extension alphabet when every character is representable; otherwise UCS2 for supported characters, including Persian). Advanced view provides manual GSM7, UCS2, or binary plus explicit `data_coding` configuration. CLI, Web UI and headless must use identical encoding-selection rules. Segment/PDU accounting must use encoded lengths, not unencoded character counts. D-032 fixes unpacked GSM7, strict BMP UCS2 and binary baseline; interoperability still requires tests.
- Concurrent `submit_sm_resp` matching by `sequence_number`, timeouts, negative statuses, and late responses. **Never blindly retry an uncertain submit.**
- **DLR-request default (D-030):** `registered_delivery=0x00` (no receipt requested). Per-run Basic Request DLR on/off switch; Advanced provides full-byte `registered_delivery` override, winning over the switch. All interfaces use one shared resolution rule. Basic ON is `0x01` (success/failure); Advanced supports `0x02` (failure-only) and explicit `0x03` (vendor-specific success-only test). SMPP 3.4 reserves `0x03`: show warning and do not promise this behavior. Receiving/ACKing DLR is independent of whether receipts were requested.
- DLR receipt via `deliver_sm`, appropriate `deliver_sm_resp`, classification, cross-session message ID correlation, duplicate/unmatched/missing/late report metrics.
- Per-run completion choice: **Submit Completion** or **Submit + DLR Completion** with configurable DLR drain deadline.

## Confirmed submit, DLR and headless details (D-032–D-034)

- **D-032:** Default `submit_sm` fields: empty `service_type`, schedule and validity, zero protocol/priority/replacement/default-message controls; UDHI set only for UDH multipart, `registered_delivery` per D-030/D-031. Strict ASCII Basic addresses never rewritten. GSM7 unpacked/DCS `0x00`, UCS2 strict BMP UTF-16BE/`0x08`, binary raw/`0x04`. Auto 16-bit UDH references, optional 8-bit UDH or SAR, max 255 segments, and 254-octet no-split `short_message` bound or explicit Advanced `message_payload`. Full details in [protocol](PROTOCOL_DATA_FAULTS.md#confirmed-submit_sm-defaults-d-032).
- **D-033:** For an incoming DLR, `message_id` is read **first from text `id:` within `deliver_sm.short_message`**, with `receipted_message_id` TLV fallback only. On conflict, the text ID wins with diagnostics recording both values. Delivery state parsing is independently TLV-first (`message_state`), text `stat:` fallback. Exact opaque cross-session ID matching and per-segment correlation are default. `0x02` conditional failure and reserved `0x03` vendor test never imply guaranteed receipts. Default configurable DLR drain **60s**. Full details in [DLR](PROTOCOL_DATA_FAULTS.md#confirmed-dlr-correlation-and-lifecycle-d-033).
- **D-034:** The noninteractive `smpp-studio run --headless --config ... --result ...` accepts versioned JSON/YAML, emits JSON `schema_version=1` and status exit codes `0/2/3/4/5/6`. C++ and other frameworks own assertions. See [headless](HEADLESS_INTEGRATION.md#confirmed-headless-v1-contract-d-034).

**These are approved design requirements, not implemented features.**

## Confirmed response-policy model (D-035)

For inbound `deliver_sm`, select exactly one client response outcome: successful `deliver_sm_resp`, configured nonzero-status `deliver_sm_resp`, or intentionally no response. Delay is an optional ACK property, not another outcome. Error plus no-response percentages are at most 100%; the remainder is successful ACK. DLR-specific policy fully overrides the generic policy. No configured policy means immediate successful ACK. Configurable seed and accurate selected-versus-emitted counters apply to all adapters. D-036 additionally fixes **Fixed delay off by default (0ms)**, independently selected for ACKs after the D-035 outcome using a separate configured percentage, configurable 60s default delay maximum, four initial error statuses (`ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`), default maximum **10,000 queued delayed ACKs per run**, and controlled `resource_limit` termination at capacity. No Response has no delayed timer. This does **not** alter the underlying SMSC delivery state or authorize implementation.

## Confirmed headless/Go integration detail (D-038)

The D-034 outer contract is refined by D-038: strict `schema_version=1` with `connection.sessions` (nonempty named bindings using env-secret references), `message`, `traffic`, `completion`, optional `capture`, `faults`, `seed`; unknown/duplicate/invalid fields rejected. No legacy `test` wrapper. One finite run, same Go config resolution for every interface. Structured JSON includes status/reason, redacted effective config, UTC timestamps, counters/per-session/DLR/diagnostics, and preserves execution-only exit codes 0/2/3/4/5/6. Go API uses `New`, `Run(context.Context)`, `Stop`, `Close` (idempotent). C++ calls the single binary via subprocess and owns assertions. Full policy: [Headless](HEADLESS_INTEGRATION.md#confirmed-nested-headless-and-go-api-v1-d-038).

## Traffic profiles — all six in V1

1. Constant TPS.
2. Ramp-up/Ramp-down.
3. Step Load.
4. Burst/Spike.
5. Unlimited (limited by available resources, in-flight bounds, and safety limits).
6. Adaptive Window: independent per-TX batch controller with D-037 3s decision deadline, severe-degradation Critical Pause, authorized automatic probe recovery, and bounded run-wide outstanding capacity.

Per-session Adaptive Window is independent. Run-wide traffic allocation is configurable: Equal, Adaptive, or Weighted. A global limiter governs capped profiles. D-037 fixes Adaptive Window behavior: initial 32/max 1,024 PDUs per TX, 3s batch deadline distinct from the 5s submit timeout, healthy p95≤1s, 12.5% minimum-1 increase, half-shrink on degradation, three severe batches to zero, 5s authorized single outstanding probe, two consecutive successful probes to restart at window=1, 8,192 shared outstanding cap, and configurable 10m finite run deadline. Probe traffic is separately observable and part of total PDUs. Detailed behavior: [Traffic Control](TRAFFIC_CONTROL.md#confirmed-adaptive-window-defaults-and-lifecycle-d-037). One primary traffic profile applies per run; distribution selects future work without silently switching algorithms. Other profile tuning remains open.

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
