# Traffic Profiles and Adaptive Window

## Six V1 profiles

| Mode | Control intent | Notes |
| --- | --- | --- |
| Constant TPS | Steady configured PDU/s | Target vs Actual plotted separately |
| Ramp-up/down | Continuously change target rate | start/end and durations configurable |
| Step Load | Discrete sustained TPS levels | step size, hold, boundaries configurable |
| Burst/Spike | Short higher-rate pulses | peak/duration/cooldown configurable |
| Unlimited | Maximum feasible sender capacity | still obey connection windows and safety/resource limits |
| Adaptive Window | Batch-based in-flight adaptation | independent controller per send-capable session |

Adaptive Window profile defaults and controller semantics are approved in D-037; numeric defaults for the other five modes remain open. Do not claim 50k TPS in every mode, particularly batch-based Adaptive Window with wait overhead.

## Confirmed Adaptive Window defaults and lifecycle (D-037)

**Approved design, not implemented.** Per TX/TRX send-capable session, apply this independently; all values configurable with validation.

| Control | Default |
| --- | --- |
| Initial window | 32 submit PDUs |
| Maximum window | 1,024 submit PDUs |
| Batch decision deadline | 3s, distinct from 5s per-`submit_sm` response timeout (D-024) |
| Healthy batch p95 | At most 1s |
| Intentional delay between batches | 0ms |
| Growth after healthy batch | Add `max(1, floor(current_window / 8))`, capped by maximum |
| Shrink after degraded batch | `max(1, floor(current_window / 2))` |
| Critical Pause threshold | 3 consecutive severely degraded batches; normal window = 0 |
| Recovery probe interval | 5s; at most one outstanding probe per zero-window session |
| Recovery requirement | 2 consecutive successful probe `submit_sm_resp`; reopen at window = 1 |
| Run-wide outstanding submit cap | 8,192 pending PDUs including probes |
| Adaptive run wall-clock deadline | 10 minutes, configurable and finite |

**Batch classification and control**

- At most one normal batch at a time per sender session. Dispatch up to the per-session window subject to pending-cap capacity, optional global TPS limit and destination authorization; wait for all the batch's sequence-matched responses until the 3s batch decision deadline. This is batch-based control, not a rolling unlimited pipeline.
- A **healthy** batch receives all successful `submit_sm_resp` (status `ESME_ROK`) before its deadline and has response-latency p95 ≤1s. Increase by the configured additive fractional step, clamped to maximum.
- A **degraded** batch has any missing response at the batch deadline, any negative `command_status`, or p95 >1s. Reduce to at least 1, unless the severe streak triggers Critical Pause.
- **Severe** degradation means any response missing by the batch deadline or a recognized SMPP 3.4 throttling/congestion status (exact supported status classification list must be documented for interoperability). Three **consecutive** severe batches set normal window to zero. A non-severe batch resets the severe streak; an ordinary non-congestion negative response or high latency alone shrinks without counting severe.
- Batch health and the window transition are decided at the **batch deadline**; late responses are still observed but cannot retrospectively make an old batch healthy or cause a second transition for the same batch.
- The **3s batch decision deadline does not replace** the approved **5s per-submit response timeout**. Missing at 3s means deadline-unresolved, not already timed out/rejected. Pending from earlier batches still consumes actual in-flight capacity until resolved or its individual timeout. Never retransmit an ambiguous submit automatically; bounded sequence ledgers and late-response tombstones distinguish old/late/unmatched responses.

**Critical Pause and controlled recovery**

- At window=0, stop **normal** submissions on that TX/TRX session; do not stop DLR receive, `deliver_sm_resp`, `enquire_link`, or normal connection health processing.
- Every 5s, permit at most **one in-flight** authorized `submit_sm` probe per zero-window sender. A probe succeeds only on `ESME_ROK` before its own per-submit timeout; timeout or error resets the consecutive success count. Two consecutive probe successes restore normal window=1, followed by gradual growth.
- Probe destination and content must be explicitly authorized; one unambiguous fixed approved destination may be explicitly selected in resolved configuration, while dynamic/multi-destination tests require an explicit probe target. Never invent or silently select a live number. Probes may incur SMSC costs.
- Probe PDUs count **both** separately in probe metrics and within total physical sent PDU TPS, global outstanding capacity and safety limits.
- The finite Adaptive run deadline defaults to **10 minutes**, even if Critical Pause prevents reaching the desired message count. Termination reason must distinguish run deadline, normal completion, manual stop and full TX loss; the simulator records execution status but makes no pass/fail assertions.

**Cross-session invariants**

- One primary traffic profile per run. Adaptive Window is not invisibly stacked with Constant/Ramp/Step/Burst/Unlimited; optional run-wide TPS cap applies to it. Existing Equal, Adaptive and Weighted session distribution selects **future work only** and must never silently substitute distribution modes. Independent session windows and run-wide pending cap still govern dispatch.
- One TX loss with another healthy TX available continues; all TX unavailable **stops** the run (D-017), overriding attempts to recover the run. All RX lost while TX healthy follows D-016 configurable Continue MT / Pause MT and in-process RX reconnect.
- Maintain per-session healthy/degraded/severe batch counters, p95, in-flight/unresolved/late counts, window transitions, Critical Pause dwell time, probe attempts/success/error, throttling and distribution allocation, with accurate aggregate metrics independent of capture sampling.
- **Design acceptance** should exercise out-of-order responses, 3s versus 5s timeouts, severe streak reset, false-positive congestion avoidance, late-response bookkeeping, one probe at a time, two successful probes, per-session isolation, TX and RX outages, run deadline, authorization and bounded resource use. No acceptance tests have been executed.

## Adaptive Window — confirmed behavior

For each active TX/TRX sender session:

1. Maintain `current_window` bounded by `0..max_window`.
2. Send one batch up to `current_window` (not an unlimited rolling pipeline).
3. Wait up to configured deadline for **all corresponding `submit_sm_resp`**; responses can arrive out of order.
4. Healthy rounds allow gradual increase; congestion, missing responses or unhealthy latency cause decrease. The exact gain/decrease tuning is now confirmed by D-037 and remains configurable.
5. Never exceed `max_window`.
6. Repeated severe degradation may put the **normal traffic window at zero** (`Critical Pause`).
7. While zero, periodically send a **single explicitly controlled `submit_sm` recovery probe** to an authorized test destination. After sufficient successful probe observations, resume a small window and grow gradually, still respecting maximum.
8. Count probe traffic separately from normal traffic and include it in all safety and actual-throughput accounting.

Probes can be accepted and charged by the SMSC: never implicitly treat them as cost-free. `enquire_link` alone cannot establish SMSC submit processing health. D-037 specifies the V1 defaults for `inter_batch_delay`, batch deadline, probe cadence and success threshold.

A request crossing a batch deadline becomes **unresolved/ambiguous**, not necessarily rejected. Keep sequence bookkeeping for late results and ensure timed-out requests do not cause excess unsafe in-flight traffic. Do not re-submit uncertain messages automatically. D-037 requires bounded pending tracking, late-response tombstones and no unsafe resend. Exact retention/expiry and connection fencing mechanics remain implementation-design details.

## Session distribution

Run configuration selects:
- **Equal**: proportional target share split equally; inability of one session to use its share can reduce Actual TPS.
- **Adaptive**: distribute future work toward healthy sessions based on observed capacity; never exceed their independent session limits or global cap.
- **Weighted**: relative target shares use user-specified positive weights (e.g. 5:3:2).
- Aggregate rate governor prevents overshoot for rate-capped patterns. Unlimited removes rate target, not memory/window/safety bounds.

How session-weight constraints interact with Adaptive distribution should be explicitly specified before code. Do not silently change a test's requested distribution mode.

## Connection and DLR interactions

- One TX disconnect: isolate it, continue via other healthy senders.
- All TX disconnected/unavailable: **STOP the run** immediately; do not wait for automatic recovery.
- All DLR receiving sessions down but TX still available: run-level switch **Continue MT** or **Pause MT** while reconnecting RX in-process. Pause stops new generation/dispatch and keeps buffers bounded. Restore sending when a suitable bound RX session returns.
- A TRX connection may serve both roles. If all send-capable sessions fail, the TX stop rule takes precedence.
- In the event of process crash/restart, old run stays Interrupted and is not resumed.

## Observability

Sample all: target/actual `submit_sm` PDU/s, accepted/negative/timed-out responses, current/max window, batch count, healthy/unhealthy rounds, late responses, outstanding requests, probe attempts/success, time in Critical Pause, throttling and error statuses, distribution allocations, per-session state, CPU/memory benchmarking notes. Collect aggregate metrics precisely even in sample-capture mode.
