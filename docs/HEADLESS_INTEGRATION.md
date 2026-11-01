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

The above is confirmed **design**, not implemented code. D-038 now confirms the nested V1 input/result categories, validation, and Go lifecycle operations in the section below; optional low-level type spellings may be refined without changing these approved meanings.

## Confirmed nested Headless and Go API V1 (D-038)

**Approved design; implementation not started.** D-038 makes the D-034 outer contract concrete without adding a service, native C++ SDK or assertion engine.

### Versioned input and validation

- JSON and YAML share one strict `schema_version: 1` and **only** the approved top-level `connection`, `message`, `traffic`, `completion`, optional `capture`, `faults`, `seed`. A legacy top-level `test` is **not** accepted. Unknown keys, duplicate YAML mapping keys, mismatched types, invalid values and incompatible field combinations cause configuration error (headless exit 2).
- `connection.sessions` is a nonempty list of named SMPP bindings. Each entry uses `id`, `host`, `port`, `system_id`, `password_env`, `bind_mode` (tx/rx/trx), optional positive `weight`, and optional timeout and enquire-link overrides inheriting D-024/D-025. One TRX entry is the usual simple-case default. No plaintext password key is accepted and credentials never enter reports.
- `message` supports fixed `source`, `destination`, `text` plus explicit binary base64 and dataset/CSV alternatives. Optional `encoding=auto`, `multipart=auto`, `request_dlr=false`, Advanced `registered_delivery` override and `data_coding` and individual source/destination TON/NPI overrides obey D-026–D-033; no silent address rewriting. Dataset options remain nested inside `message`.
- `traffic` contains `mode` (one of six V1 profiles), finite `count`, `duration` or finite Stop-at-End dataset, mode-specific fields (for example `tps`/`adaptive`), and `distribution` (equal/adaptive/weighted). Do not permit accidental unbounded work. D-037 provides Adaptive's bounded defaults, including a default 10-minute run deadline.
- `completion` contains `mode=submit|submit_and_dlr` and `dlr_timeout` (60s default from D-033). Optional `capture` is bounded and `faults` contains D-035/D-036 ACK policies. Optional integer `seed` allows reproducible selections; report the resolved seed without exposing secrets.
- Relative input dataset paths resolve **from the config file directory**, not the current working directory. Output is controlled solely by the `--result` argument. All modes use the same typed config resolution and validation.

### Versioned result and exit behavior

- The required top-level JSON keys are `schema_version`, `run_id`, `run_status`, `termination_reason`, `started_at`, `finished_at`, redacted `effective_config`, `counters`, `per_session`, `dlr`, `diagnostics`. Timestamps are UTC RFC3339; counters are integral; durations and latency fields expose explicit **milliseconds** units. On failure preserve available partial counters and attempt atomic result publication; logs go to **stderr**, not into machine-readable output.
- `run_status`: `completed`, `stopped`, `cancelled`, `error`, and historical persisted `interrupted` after process crash. Distinguish `termination_reason`: `completed`, `dataset_exhausted`, `all_tx_unavailable`, `resource_limit`, `run_deadline`, `cancelled`, `startup_failure`, `internal_error`; additional bounded diagnostic detail may be recorded.
- Keep D-034 exit code mapping **unchanged**: 0 normal finite completion even on negative SMSC responses/missing DLR; 2 config; 3 startup/connect/bind; 4 runtime/all-TX/resource limit; 5 cancelled or run deadline; 6 internal/result write fault. Result write failures report through stderr and exit 6. Caller frameworks own assertions.
- Report distinct logical SMS and physical submit PDU counters, per-segment acceptance/rejection/timeout/uncertainty, recovery probes, DLR delivered/failed/missing/duplicate/unmatched/ambiguous/late, **text-first DLR message-ID conflicts** and selected-versus-sent ACK faults. Label event counters that overlap to avoid double counting; full capture is optional and bounded.

### Language and lifecycle interfaces

- Go public lifecycle design: `New(Config) (Runner,error)`, `Runner.Run(context.Context) (Result,error)`, `Runner.Stop()`, `Runner.Close()` (idempotent). The concrete Go type form may be made idiomatic without changing these four operations and data/error semantics. No extra engine or built-in assertion DSL. Caller cancellation and shutdown clean up network and local resources; never resend an uncertain submit.
- C++ (Google Test/Catch2) starts the **same** `smpp-studio run --headless --config ... --result ...` with safe argument arrays, isolated temporary paths, environment credentials and a process deadline, then parses JSON and owns its own assertions. Python/Java/shell can use the same contract.
- Never expose secrets in logs or result files. This design does **not** authorize implementation.

## Illustrative invocation (not yet implemented)

```sh
smpp-studio run --headless --config case.yaml --result result.json
```

```yaml
# Design example consistent with the approved D-038 groups; not yet executable.
schema_version: 1
connection:
  sessions:
    - id: main-trx
      host: 127.0.0.1
      port: 2775
      bind_mode: trx
      system_id: test-client
      password_env: SMPP_PASSWORD
message:
  source: "1000"
  destination: "989121234567"
  text: "Test {{sequence}}"
  request_dlr: true
traffic:
  mode: constant
  count: 100
  tps: 50
  distribution: equal
completion:
  mode: submit_and_dlr
  dlr_timeout: 60s
```

```json
{
  "schema_version": 1,
  "run_id": "example-001",
  "run_status": "completed",
  "termination_reason": "completed",
  "started_at": "2026-11-01T00:00:00Z",
  "finished_at": "2026-11-01T00:00:02Z",
  "effective_config": {"redacted": true},
  "counters": {"logical_messages": 100, "submit_pdus_sent": 100, "submit_resp_ok": 100},
  "per_session": [],
  "dlr": {"delivered": 100, "missing": 0},
  "diagnostics": []
}
```

The invocation, outer schema and exit code meanings are **confirmed in D-034**. The examples illustrate the now approved D-038 schema structure; precise optional advanced field/type spellings remain a normal schema-design and implementation task. The runner must keep logs separate from JSON output; use stable versioned machine-readable results, atomic result-file writes when possible, and redact credentials.

## DLR request contract (D-030–D-033)

Basic DLR toggle: OFF `0x00`, ON `0x01`. Advanced values: `0x02` failure-only and `0x03` vendor-specific success-only compatibility probe (**reserved in SMPP 3.4**). Preserve the raw byte, expose a compatibility warning, and report actual behavior for C++ framework assertions; simulator has no assertions. This does not add SMPP v5 support.

## Resource-limit outcome (D-036)

If delayed `deliver_sm_resp` pending-ACK capacity reaches the configured limit (default 10,000), stop with structured `termination_reason=resource_limit`, preserve partial observations and use D-034 **exit code 4** (runtime could not complete). This is an execution/resource condition, **not** an in-simulator test assertion or an SMSC delivery status; the invoking C++/Go/Python/Java test framework owns the verdict. No new headless service or exit code is introduced.

## D-039 result handling

Headless writes its JSON result to the explicit `--result` location only; optional capture artifacts must be bounded and referenced from the result. Local capture defaults to High Performance with 1,000 samples and a 512 MiB raw limit; report snapshots default to 5s. A restarted process must not resume an interrupted Run. Returned TX binds can rejoin an active Run as a new generation for future traffic only. Startup all-TX bind failure, runtime all-TX loss and correlation `resource_limit` retain their distinct established execution outcomes.

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

D-038 closes the minimum nested input/output groups, lifecycle methods and strict validation. Remaining engineering: optional Advanced-field type spellings, storage layout and deterministic ordering/cancellation tests, preserving D-034. D-033 DLR message ID must be resolved text-first consistently in every adapter.
