# Acceptance and Validation Plan (D-040 approved; not yet executed)

This is a proposal for verifying the agreed scope. **No implementation or test results exist yet.**

## Functional/PDU correctness

- Bind TX/RX/TRX over TCP and validate success, invalid credentials, timeout, unbind, and connection cleanup against an isolated test server.
- Send one `submit_sm`, correlate its `submit_sm_resp` by `sequence_number`, parse negative statuses, late responses and out-of-order replies.
- Validate length/encoding/TON/NPI/TLV/UDH/SAR cases with fixtures and a known-good interoperable SMSC.
- Verify Auto TON/NPI separately for clear alphanumeric and explicitly international source/destination formats, safe behavior for ambiguous numeric/short-code addresses, independent explicit source/destination TON/NPI overrides, and **byte-for-byte unchanged configured address strings** after classification. Compare resolved metadata between CLI, UI, Go API and headless.
- Verify D-029 specifically: `MyBrand` -> `5/0`, `+989121234567` -> `1/1`, `989121234567` and `1000` -> `0/0`; test separately for source/destination and preserve literal source/destination address bytes (including `+`).
- Validate default Auto multipart using UDH, manual UDH and SAR TLV methods, and No automatic splitting, including short message bypass, exact sequence/reference fields, encoded boundaries, partial segment `submit_sm_resp` failures and explicit non-truncating oversize handling. Confirm distinct counters for logical messages and actual submitted segments/PDUs.
- Verify Auto encoding selects GSM7 for GSM 03.38 default/extension characters (including escaped extension characters) and UCS2 for supported Persian/non-GSM text; verify manual GSM7/UCS2/binary and `data_coding` overrides; never silently lose unsupported Unicode. Verify encoded byte/segment accounting and SMPP default-alphabet interoperability.
- Verify `registered_delivery=0x00` as default, per-run DLR opt-in, exact full-byte Advanced override, and parity across CLI/Web UI/Go/headless. Validate that a Submit+DLR run with no expected DLRs does not hang awaiting them.
- Verify DLR OFF `0x00`, Basic ON `0x01` and Advanced `0x02`; permit deliberately sending raw `0x03` with a visible SMPP 3.4 reserved-value warning. Confirm no automatic byte rewriting and report observed SMSC behavior rather than assuming success-only support.
- Concurrent DLR intake while submit load runs, correlation across TX and RX sessions, duplicate/unmatched/missing/late receipts, and `deliver_sm_resp`.
- Verify intentional DLR ACK error, delay and no-response distribution with deterministic seed; keep actual counts.
- Verify no MO test scenario is required, no TLS dependency, and no query/cancel/replace operation in V1.

## Approved D-032–D-034 acceptance cases (not yet executed)

- **D-032:** inspect actual bind/submit defaults, literal unchanged Basic addresses and invalid input rejection; GSM7 unpacked octets/DCS `0x00`, strict BMP UTF-16BE/`0x08`, raw binary/`0x04`, GSM extension septets, non-BMP rejection, 16-bit/8-bit UDH vs SAR framing, 255-part cap, no-split at or below 254 `short_message` octets and explicit `message_payload` behavior above it.
- **D-033:** DLR with both text `id:` and TLV: **text ID must win** and mismatch is reported with both values. DLR with unusable/missing text ID: TLV fallback; delivery state resolution remains `message_state` TLV-first and text `stat:` fallback. Check opaque exact case-sensitive matching, cross-session receipt, duplicate/ambiguous IDs, partial multipart segment acceptance, conditional `0x02` and vendor `0x03` no false Missing, default configurable 60s drain.
- **D-034:** versioned JSON/YAML config parity, reject unknown keys and unbounded unintended workloads, no headless listener/UI, secret redaction, C++ subprocess test + cleanup, JSON result on execution failures where possible, aggregate accuracy without full capture, and exit statuses `0/2/3/4/5/6` without built-in assertions.

These are **acceptance requirements**, not test results.

## D-035 acceptance requirements (not executed)

Verify exactly one selected response per received `deliver_sm`; unconfigured policy gives immediate `ESME_ROK`; configured error plus no-response percentages never exceed 100%; remaining probability produces success; DLR override replaces generic policy in full, without percentage blending; delays apply only to ACK outcomes; configured random seed allows repeatable selection inputs; no-response produces no delayed ACK; per-policy selected and actually emitted counters remain accurate even with sampling off, and injected response outcomes do not alter delivery status. D-036 already fixes delay defaults, four initial error statuses and delayed-ACK capacity; these are confirmed acceptance requirements.

## D-036 acceptance requirements (not executed)

Verify Fixed ACK delay disabled at 0ms by default, selectable only after success/error ACK outcome via an independent configured delay percentage, maximum delay 60s by default but configurable, and exactly the four initial supported nonzero statuses. Verify selected No Response creates neither timer nor later ACK. With a bounded pending-ACK limit of 10,000 by default, reaching capacity must stop the run with `termination_reason=resource_limit` (runtime exit code 4 per D-034), preserving selected-vs-emitted counts and partial evidence rather than silently converting or dropping scheduled outcomes. The cap applies to pending delayed ACKs per run, not total DLR count. These are future acceptance cases, not results.

## Traffic/controller correctness

- Constant, Ramp-up/down, Step, Burst, Unlimited, Adaptive Window each produce correct timing intent and stable observability.
- Under health: Adaptive Window increases without exceeding configured max; under missing/late/congestion responses it reduces and eventually can reach zero.
- At zero: no regular traffic; only bounded authorized `submit_sm` probes; auto-recover after configured successes.
- Independent session windows with global capped rate, Equal/Adaptive/Weighted distribution.
- Accurate accounting of sent PDUs, accepted responses, pending/uncertain and actual TPS; no duplicate resend of an uncertain submit.
- Ensure in-flight and capture limits prevent unbounded memory usage.

## D-038 Headless/Go API acceptance requirements (not executed)

- Compare identical JSON/YAML configurations with `schema_version=1`, named `connection.sessions`, explicit fixed messages, finite traffic, completion and seed. Reject legacy `test` top-level, unknown fields, duplicate YAML keys, unsupported enum/types, missing/empty sessions and accidental unbounded runs with configuration exit 2 and diagnostics.
- Validate environment-referenced secrets only, no plaintext config/result/log leakage, one TRX default, RX/TX separated sessions, per-session timers, relative dataset path resolution against the config directory, explicit caller result path and no UI/network listener in headless.
- Validate UTC RFC3339 time fields, stable millisecond units, redacted effective config, integer logical/physical/per-session/DLR/ACK-fault counters, bounded detail samples and accurate aggregates when capture disabled. Failure must still attempt a structured partial JSON result; output write failure uses exit 6.
- Exercise exact D-034 codes 0/2/3/4/5/6 independently of SMSC negative statuses and test assertions. Confirm `run_status`/`termination_reason` values and C++ test subprocess integration with safe arguments, deadline, cancellation cleanup.
- Validate Go lifecycle `New`, `Run(context.Context)`, `Stop`, idempotent `Close`; CLI/Web/headless reuse a common engine with no automatic resend or global mutable per-run singleton. These tests are **planned**, not run.

## D-037 Adaptive Window acceptance requirements (not executed)

- For each TX/TRX, verify initial=32, max=1,024, independent window state, healthy p95<=1s, 3s batch decision deadline separate from the fixed 5s per-PDU submit-response timer, 0ms inter-batch delay, increase by max(1,floor(window/8)) and degrade to max(1,floor(window/2)).
- Show three **consecutive severe** rounds cause window zero; ordinary negative non-throttling statuses or p95 degradation reduce but do not increment severe streak. Test a non-severe round resetting the severe streak, including a batch with out-of-order responses.
- Pending at 3s must not be miscounted as 5s submit timeout; late responses cannot retroactively change old batch window decisions or match a later request; no automatic retransmission of ambiguous submit PDUs. Outstanding requests from old batches consume per-session and global capacity until resolved/individual timeout.
- At zero, no normal submit dispatch; at most one authorized recovery probe outstanding every 5s per session, two consecutive successful in-time responses reopen at window=1; bad/expired probe resets streak. Probes count in physical TPS, safety budgets and separate metrics, may be chargeable, and cannot use unapproved live destinations.
- Shared global cap=8,192 pending PDUs, capped-rate and independently selected Equal/Adaptive/Weighted distribution parity, one primary traffic profile, TX-one-loss continuation, all-TX-loss stop, RX-only-loss Continue/Pause, uninterrupted DLR ACK handling while paused, and bounded finite Adaptive default run deadline=10m (configurable). Distinguish run deadline from completed normal target and emit the appropriate D-034 execution reason/exit code. No performance PASS claim from these design cases.

## D-039 planned verification (not executed)

Verify independent TX reconnect generations, RX outage Continue/Pause, 120-second DLR identity retention, default one-million mapping cap, local bounded reports, capture truncation, and five-second snapshots without automatic run replay. These are design requirements, not executed tests.

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

## Confirmed V1 acceptance and benchmark gate (D-040)

**Design acceptance criteria confirmed 2026-11-01; no code has been implemented and no tests or benchmarks have been run.** An item is PASS only after evidence from an actual executed check; design approval is not test PASS.

### Functional and cross-language gates

- All specified SMPP 3.4 bind/submit encoding, segmentation, TON/NPI, timeout and negative-status behaviors; D-033 text-first DLR identity, TLV-first delivery state, optional receipt expectations, across-session/per-segment correlation; D-035/D-036 faults; all six traffic modes including D-037 recovery; D-039 reconnect, map capacity, capture and interrupted reports.
- Automatic tests must demonstrate safe stop, no blind retry or resume, redaction, bounded resources, authorized probe destinations and no accidental live traffic.
- JSON/YAML strict schema-v1 parity, Go `New/Run/Stop/Close` lifecycle, one headless executable invoked from C++ Google Test or Catch2, stable structured JSON and exit codes. No embedded test assertions or mandatory second service.
- One local scripted acceptance command and reports are preferred; no GitHub Actions requirement. Functional checks cannot be replaced by a screenshot or narrative.

### Measurable performance and evidence

| Dimension | Confirmed target/reference |
| --- | --- |
| Throughput metric | **Actual physical `submit_sm` PDUs transmitted/s**, not logical SMS or configured TPS |
| Reference host | Linux x86_64; at least 16 logical CPUs, 32 GiB RAM |
| Network and peer | Loopback or isolated >=1 GbE; independently measured SMPP test peer >60k submit PDUs/s |
| Scenario | 8 TX/TRX sessions, short fixed GSM7, DLR OFF, High Performance without optional raw capture |
| Measured duration | 5 minutes steady throughput with 1-second series retained |
| PASS performance objective | Actual average >=50,000 submit PDUs/s for 5 minutes; no unexplained PDU loss or forced resource stop; peak simulator RSS <8 GiB |
| Evidence | At least two reproducible runs, host/OS/kernel/Go/CPU, peer capacity, connections, RTT, TPS samples, CPU/RSS/GC, latency and timeout/error counts |

An underpowered/unmeasured peer cannot establish simulator 50k PASS. A benchmark falling short is **not verified/failed**, not silently declared PASS. Batch Adaptive mode is separately measured and is **not** required to reach 50k TPS, since it has a different pacing model.

**Separate DLR load:** 5,000 submit PDUs/s for 3 minutes against a deterministic test peer with known IDs/final receipts; validate exact correlations, ACK/counter correctness, late/unmatched categories and bounded resource use. This does not claim 50k DLR/s.

### Security, distribution and phase gate

- Every run requires a finite count/duration/Stop-at-End dataset, explicitly authorized destination(s), and explicit authorized Adaptive probe target; safe resource stops and potential SMSC charging visibility.
- Embedded UI listens only on localhost by default. Remote binding and access policies are excluded from automatic exposure until separately reviewed. Secrets through environment and redacted logs/JSON; sanitized local reports and conservative filesystem permissions.
- One binary plus importable Go package; smoke builds for Linux amd64 and Windows amd64 are the initial release checks. No mandatory broker/database/orchestrator or GitHub Actions.
- **Design freeze** occurs only after consistency review confirms D-001–D-040, no unresolved P0 product behavior and cross-document agreement. The design freeze **does not authorize implementation** (D-023) and supplies **no evidence** of completed functional/performance acceptance. A future release requires actually executed tests and benchmark evidence or explicit owner-approved performance exception.

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
