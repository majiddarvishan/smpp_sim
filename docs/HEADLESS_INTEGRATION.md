# Headless Integration Testing — Go, C++, Python, Java

## Accepted interaction model

One shared Go engine supports:
1. An importable **Go API** with explicit config, context cancellation, session lifecycle, send/wait, and result structs.
2. A **non-interactive headless runner** in the same executable for tests in **C++** and any other language able to start a child process.

Headless invocation requires **no Web UI, no running web service, and no interactive CLI workflow**. The harness provides a JSON or YAML file/config path and receives **structured JSON** results. No C++ native library, CGO, persistent API server, or external orchestrator is required.

The **caller owns assertions** (Google Test / Catch2 in C++, Go testing, pytest, JUnit, etc.). The simulator must **not** implement a built-in assertion DSL, thresholds to decide test success, or a generic test-management engine. Simulator reports execution outcomes and protocol observations, not business pass/fail.

## Confirmed headless V1 contract (D-034)

- One existing executable: `smpp-studio run --headless --config <path.json|path.yaml> --result <path.json>`. No UI listener, interactive prompts, service, native C++ SDK, separate Go engine, or built-in assertions.
- Use `schema_version: 1` for both input and result. Require explicit target host, port, system ID, credential from environment (e.g. `password_env`), source/destination for fixed messages, and **finite termination** (count, duration, or finite Stop-at-End dataset). No implicit unlimited sends. Reject unknown config keys and impossible combinations with structured diagnostics; JSON/YAML must resolve to identical effective settings.
- Confirmed config top-level groups: `schema_version`, `connection`, `message`, `traffic`, `completion`, optional `capture`, `faults`, `seed`. Reuse the Go engine config; the basic run defaults inherit D-024 through D-031. Provide `--result` as separate output location, not two competing settings.
- Result JSON always attempted (including failed start), written atomically where feasible. Top-level: `schema_version`, `run_id`, `run_status`, `termination_reason`, `started_at`, `finished_at`, `effective_config` **redacted**, `counters`, `per_session`, `dlr`, `diagnostics`. Stable numeric counters include logical SMS attempted, physical PDUs sent, submit OK/rejected/timeout/uncertain, segment outcomes and DLR receipt categories. Bounded detail samples only; total counters remain accurate when sampling is off. Never emit credential values.
- Confirmed process exit codes express execution health **not test assertions**: `0` = run finished as requested even with SMSC rejections/missing DLR; `2` = configuration/input error; `3` = startup/connect/bind failure; `4` = runtime could not complete, including all-TX loss or safety stop; `5` = cancelled/deadline; `6` = internal or result-write error. The result's termination reason is more detailed than an exit code. Caller (C++ Google Test/Catch2, Go, Python, Java) decides test pass/fail from the JSON.
- Deterministic input generation uses explicit seed; timing/network scheduling and remote SMSC outcomes are not promised bit-for-bit deterministic. Preserve CLI/Go/headless semantic parity.

The above is confirmed **design**, not implemented code. Nested config fields and Go API method signatures must still be specified while keeping the approved outer schema and exit codes.

## Illustrative invocation (not yet implemented)

```sh
smpp-studio run --headless --config case.yaml --result result.json
```

```yaml
# Illustrative draft; not an approved config schema
schema_version: 1
connection:
  host: 127.0.0.1
  port: 2775
  bind_mode: trx
  system_id: test-client
  password_env: SMPP_PASSWORD
test:
  count: 100
  timeout: 30s
  traffic: {mode: constant, tps: 50}
message:
  source: "1000"
  destination: "989121234567"
  text: "Test {{sequence}}"
  request_dlr: true
completion: {mode: submit_and_dlr, dlr_timeout: 10s}
```

```json
{
  "schema_version": 1,
  "run_id": "example-001",
  "run_status": "completed",
  "submit_pdus_sent": 100,
  "submit_resp_ok": 100,
  "submit_resp_error": 0,
  "submit_resp_uncertain": 0,
  "dlr_delivered": 100,
  "dlr_missing": 0
}
```

The invocation, outer schema and exit code meanings are **confirmed in D-034**. Example nested fields and simplified result shape remain illustrative until nested keys/types are approved. The runner must keep logs separate from JSON output; use stable versioned machine-readable results, atomic result-file writes when possible, and redact credentials.

## DLR request contract (D-030–D-033)

Basic DLR toggle: OFF `0x00`, ON `0x01`. Advanced values: `0x02` failure-only and `0x03` vendor-specific success-only compatibility probe (**reserved in SMPP 3.4**). Preserve the raw byte, expose a compatibility warning, and report actual behavior for C++ framework assertions; simulator has no assertions. This does not add SMPP v5 support.

## C++ integration contract

The C++ test uses its usual test framework:
1. Launch the headless executable via safe process arguments with its own deadline.
2. Set config path, temporary result path, and credentials through environment or other safe means.
3. Wait for process termination; parse JSON; assert on fields of interest with Google Test/Catch2.
4. Remove test artifacts and ensure cleanup, including forcibly stopping a timed-out process.

No shell interpolation of untrusted values. The simulator should honor termination signals and clean up sockets/child work. A single integration test should not require initializing the bulk engine or listening web port.

## Test-engine design requirements

- Deterministic seeded data generation and selected fault injection.
- Shared Auto/Manual TON/NPI resolution as the Go and human interfaces; manual source and destination override independently. The configured addresses must not be silently normalized; structured diagnostics should expose effective TON/NPI without requiring full PDU capture.
- Per-run isolation, no global mutable singleton state, bounded queues, idempotent Close/Stop and goroutine cleanup.
- Explicit outcome status for normal completion, all-TX-loss stop, cancellation, runtime failure, and after-crash Interrupted (for historical run artifacts).
- No implicit retry of ambiguous submits and no replay after process restart.
- Structured error categories and useful counts; report timeouts distinctly from negative `command_status`.
- No network-facing test scenario server, native SDK or built-in assertions required.

## Pending

Finalize exact **nested** JSON/YAML keys/types, public Go API signatures, local persistence paths and deterministic ordering guarantees, all preserving D-034. D-033 DLR message ID must be resolved text-first consistently in every adapter.
