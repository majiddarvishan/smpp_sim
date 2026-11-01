# Acceptance and Validation Plan (proposed, not yet executed)

This is a proposal for verifying the agreed scope. **No implementation or test results exist yet.**

## Functional/PDU correctness

- Bind TX/RX/TRX over TCP and validate success, invalid credentials, timeout, unbind, and connection cleanup against an isolated test server.
- Send one `submit_sm`, correlate its `submit_sm_resp` by `sequence_number`, parse negative statuses, late responses and out-of-order replies.
- Validate length/encoding/TON/NPI/TLV/UDH/SAR cases with fixtures and a known-good interoperable SMSC.
- Concurrent DLR intake while submit load runs, correlation across TX and RX sessions, duplicate/unmatched/missing/late receipts, and `deliver_sm_resp`.
- Verify intentional DLR ACK error, delay and no-response distribution with deterministic seed; keep actual counts.
- Verify no MO test scenario is required, no TLS dependency, and no query/cancel/replace operation in V1.

## Traffic/controller correctness

- Constant, Ramp-up/down, Step, Burst, Unlimited, Adaptive Window each produce correct timing intent and stable observability.
- Under health: Adaptive Window increases without exceeding configured max; under missing/late/congestion responses it reduces and eventually can reach zero.
- At zero: no regular traffic; only bounded authorized `submit_sm` probes; auto-recover after configured successes.
- Independent session windows with global capped rate, Equal/Adaptive/Weighted distribution.
- Accurate accounting of sent PDUs, accepted responses, pending/uncertain and actual TPS; no duplicate resend of an uncertain submit.
- Ensure in-flight and capture limits prevent unbounded memory usage.

## Failure/lifecycle

- One TX lost while others remain: continue run with remaining sessions.
- All TX lost: terminate Test Run; never automatically resume after rebind.
- All DLR RX lost: both configured Continue MT and Pause MT policies, RX auto-reconnect in the same process, no unbounded buffered work.
- Restart/crash: existing run marked Interrupted and preserved evidence only; old run never automatically resumes.
- Dataset exhaustion stop/loop and sequential/shuffle work with deterministic seed and large CSV without full-memory preload.

## Headless integration

- Go API exercises a minimal session without CLI or Web server.
- C++ harness (e.g. Google Test) launches headless binary, reads JSON, asserts itself. Repeat with a Python/Java script as interoperability smoke if appropriate.
- JSON/YAML input parity, consistent structured results, bounded timeouts and cancellation, secret redaction and cleanup.
- Built-in assertion DSL is absent; framework determines test pass/fail.

## Performance

- Benchmark the claimed **goal** of 50,000 `submit_sm` PDUs/s on an explicitly documented, isolated host and SMPP test target.
- Record CPU/cores, RAM, OS/network, socket count, session count, window sizes, message encoding/size, DLR rate, capture mode, achieved TPS, latency distribution, sustained duration, dropped/captured counts.
- Test single-message mode separately from stress mode; measure load profiles individually, including batch adaptive overhead.
- No benchmark result may be labeled PASS without measured evidence; a design goal is not a guarantee.
- Measure whether logging/capture affects throughput and correctness; use stress/race/leak tests.

## Engineering quality

- Go unit/integration tests plus local scripted acceptance; no requirement for GitHub Actions in customer-facing repositories.
- Minimal dependencies, one-process startup, no mandatory external persistence/network service other than the intended test SMSC.
- Document cancellation, cross-platform builds, local report formats, crash evidence limits, and controlled authorized test destinations.
