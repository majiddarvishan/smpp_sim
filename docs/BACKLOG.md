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

## Candidate D-037 — Adaptive Window parameters (PENDING OWNER APPROVAL)

**Not confirmed.** This is one review package; preserve D-001–D-036 and the design-only requirement.

### Proposed default parameters

| Parameter | Candidate default |
| --- | --- |
| Initial window, per TX/TRX | 32 submit PDUs |
| Maximum window, per TX/TRX | 1,024 submit PDUs |
| Batch deadline | 3s, independent of the confirmed 5s per-submit response timeout |
| Healthy batch p95 response latency | At most 1s |
| Inter-batch intentional delay | 0ms |
| Healthy growth | Add max(1, floor(window / 8)), capped at max window |
| Degraded reduction | max(1, floor(window / 2)) |
| Critical Pause | After 3 consecutive *severely* degraded batches, window=0 |
| Recovery Probe | Every 5s, maximum one outstanding per zero-window TX session |
| Recovery success requirement | 2 consecutive successful probe submit responses; reopen window=1 |
| Global cap | 8,192 pending submit PDUs, including probes |
| Adaptive run wall-clock deadline | Default 10 minutes, configurable |

These are **candidate product defaults**, not tested or approved performance guarantees. The 50k submitted PDU/s benchmark target is separate from what this batch mode can attain.

### Batch behavior and definitions

- Each send-capable TX/TRX session has independent window, batch controller and sequence-number response tracking; select exactly one primary traffic profile per run. The existing Equal/Weighted/Adaptive session-distribution choice only determines *future* work and may not silently switch modes. An optional global TPS ceiling applies to Adaptive as well as other profiles.
- Send up to the current per-session window, respecting per-session actual outstanding capacity and the shared 8,192-pending safety cap; wait for all responses until the independent 3s batch deadline. Batches are distinct: this is not an unrestricted rolling window.
- **Healthy:** all corresponding submit responses arrive before the batch deadline with success status ESME_ROK, and observed p95 is at most the configured 1s threshold. Grow window by max(1, floor(window / 8)), up to max.
- **Degraded:** a missing response at batch deadline, any nonzero submit response status, or p95 over threshold. Shrink to max(1, floor(window / 2)), unless the Critical Pause threshold is reached.
- **Severely degraded:** missing responses at the batch deadline or recognized SMPP 3.4 throttling/congestion command statuses. Three consecutive severe batches set the normal window to zero. Non-severe batches reset the severe counter; non-congestion errors still reduce the window but alone cannot trigger zero.
- Status classification and batch health are frozen at that batch's decision deadline. A later response remains observable but must never retroactively turn that batch healthy.

### Timeout, ledger and Recovery Probe

- The 3s **batch deadline is NOT the 5s per-PDU submit timeout**. A request missing the batch deadline is deadline-unresolved, not yet a confirmed timeout or rejection. It becomes timeout/uncertain only under the independent per-submit timer; no automatic retransmission.
- A bounded sequence-number ledger must distinguish late, unmatched and unresolved submissions. Pending requests from an earlier batch occupy per-session/global in-flight capacity until resolved or their own timeout, even if the next batch starts. Late-response tombstones must be bounded; do not allow expired entries to create unlimited memory use.
- Zero-window halts normal submit traffic on that session but not DLR processing/ACK or live connection maintenance. A Recovery Probe submits one authorized test message per 5s **only if no earlier probe is still outstanding**. Successful means an ESME_ROK submit response within its own submit timeout; error or timeout resets the success streak. After two successive successful probes, reset normal window to 1; resume gradual growth.
- **Probe destination safety:** use an explicitly authorized test destination. A single unambiguous authorized fixed destination can be explicitly adopted in the effective config; multi-destination/dynamic datasets require a separately configured authorized probe target. No invented/random real-number probe; probes may incur SMSC cost. Count probes separately and also within physical PDU throughput, safety limits, and pending cap.
- Adaptive runs need a finite wall-clock deadline (proposed default 10 minutes), even if their requested message count cannot be reached during prolonged Critical Pause. Terminal reason must distinguish deadline exhaustion, full TX loss, manual stop and normal completion; these are not simulator assertions.

### Existing lifecycle rules still take precedence

- One failed TX: continue on other bound TX sessions without blind retry. All TX unavailable: terminate Test Run immediately, never auto-resume it (D-017). RX-only loss obeys the configured Continue MT/Pause MT policy with RX reconnect (D-016).
- Maintain separate accurate counts for batches, per-session windows, severe streak, probes, late responses, throttling, outstanding counts, latencies, aggregate PDU TPS, and termination reason regardless of detail capture sampling.
- Design acceptance should test out-of-order and late responses, 3s versus 5s deadlines, three consecutive severe rounds, non-congestion errors, probe single-flight and two successes, session isolation, cap enforcement, all-TX loss, RX loss, authorized targets, and finite termination. No tests or implementation are claimed.

**Next owner action:** accept or revise D-037 as one package. Only after acceptance update DECISIONS.md with CONFIRMED and move detail to TRAFFIC_CONTROL.md, PRODUCT_SPEC.md, ACCEPTANCE.md and .codex. Do not silently change the previous decisions.
