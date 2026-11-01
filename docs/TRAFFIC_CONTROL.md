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

Profile parameter names/default numbers have not been approved. Do not claim 50k TPS in every mode, particularly batch-based Adaptive Window with wait overhead.

## Adaptive Window — confirmed behavior

For each active TX/TRX sender session:

1. Maintain `current_window` bounded by `0..max_window`.
2. Send one batch up to `current_window` (not an unlimited rolling pipeline).
3. Wait up to configured deadline for **all corresponding `submit_sm_resp`**; responses can arrive out of order.
4. Healthy rounds allow gradual increase; congestion, missing responses or unhealthy latency cause decrease. The exact gain/decrease tuning remains configurable and unapproved.
5. Never exceed `max_window`.
6. Repeated severe degradation may put the **normal traffic window at zero** (`Critical Pause`).
7. While zero, periodically send a **single explicitly controlled `submit_sm` recovery probe** to an authorized test destination. After sufficient successful probe observations, resume a small window and grow gradually, still respecting maximum.
8. Count probe traffic separately from normal traffic and include it in all safety and actual-throughput accounting.

Probes can be accepted and charged by the SMSC: never implicitly treat them as cost-free. `enquire_link` alone cannot establish SMSC submit processing health. Optional `inter_batch_delay`, deadline, probe cadence and success threshold need exact V1 defaults.

A request crossing a batch deadline becomes **unresolved/ambiguous**, not necessarily rejected. Keep sequence bookkeeping for late results and ensure timed-out requests do not cause excess unsafe in-flight traffic. Do not re-submit uncertain messages automatically. The precise lifecycle/expiry of outstanding entries and connection fencing is an open engineering detail.

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
