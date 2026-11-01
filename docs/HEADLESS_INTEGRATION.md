# Headless Integration Testing — Go, C++, Python, Java

## Accepted interaction model

One shared Go engine supports:
1. An importable **Go API** with explicit config, context cancellation, session lifecycle, send/wait, and result structs.
2. A **non-interactive headless runner** in the same executable for tests in **C++** and any other language able to start a child process.

Headless invocation requires **no Web UI, no running web service, and no interactive CLI workflow**. The harness provides a JSON or YAML file/config path and receives **structured JSON** results. No C++ native library, CGO, persistent API server, or external orchestrator is required.

The **caller owns assertions** (Google Test / Catch2 in C++, Go testing, pytest, JUnit, etc.). The simulator must **not** implement a built-in assertion DSL, thresholds to decide test success, or a generic test-management engine. Simulator reports execution outcomes and protocol observations, not business pass/fail.

## Illustrative invocation (not yet implemented)

```sh
smpp-studio run --headless --config case.yaml --result result.json
```

```yaml
# Illustrative draft; not an approved config schema
version: 1
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

All field names, JSON schema, run status taxonomy, command flags, and exit codes in examples above are **proposals awaiting review**. The runner must keep logs separate from JSON output; use stable versioned machine-readable results, atomic result-file writes when possible, and redact credentials.

## C++ integration contract

The C++ test uses its usual test framework:
1. Launch the headless executable via safe process arguments with its own deadline.
2. Set config path, temporary result path, and credentials through environment or other safe means.
3. Wait for process termination; parse JSON; assert on fields of interest with Google Test/Catch2.
4. Remove test artifacts and ensure cleanup, including forcibly stopping a timed-out process.

No shell interpolation of untrusted values. The simulator should honor termination signals and clean up sockets/child work. A single integration test should not require initializing the bulk engine or listening web port.

## Test-engine design requirements

- Deterministic seeded data generation and selected fault injection.
- Per-run isolation, no global mutable singleton state, bounded queues, idempotent Close/Stop and goroutine cleanup.
- Explicit outcome status for normal completion, all-TX-loss stop, cancellation, runtime failure, and after-crash Interrupted (for historical run artifacts).
- No implicit retry of ambiguous submits and no replay after process restart.
- Structured error categories and useful counts; report timeouts distinctly from negative `command_status`.
- No network-facing test scenario server, native SDK or built-in assertions required.

## Pending

Finalize versioned JSON/YAML schema, exact Go public API, exit codes (must distinguish execution errors from test framework assertions), local persistence/result paths, and deterministic ordering guarantees. Avoid introducing different engines for different interfaces.
