# Acceptance and Validation Plan (proposed, not yet executed)

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

Verify exactly one selected response per received `deliver_sm`; unconfigured policy gives immediate `ESME_ROK`; configured error plus no-response percentages never exceed 100%; remaining probability produces success; DLR override replaces generic policy in full, without percentage blending; delays apply only to ACK outcomes; configured random seed allows repeatable selection inputs; no-response produces no delayed ACK; per-policy selected and actually emitted counters remain accurate even with sampling off, and injected response outcomes do not alter delivery status. Exact delay ranges, allowed error statuses and queue capacity remain open design topics.

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
