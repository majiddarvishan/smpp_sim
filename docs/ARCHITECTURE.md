# Architecture and Runtime Lifecycle

## Minimal composition

```text
                  CLI       Embedded Web UI      Headless runner
                    \             |                  /
                     \            |                 /
                       Shared public Go simulator API
                                  |
                SMPP engine / sessions / traffic / DLR / metrics
                                  |
                        Target SMSC (TCP)
```

The manual tool is packaged as **one Go executable** with embedded UI assets; headless mode does not start an HTTP server. The Go simulator API should be independently importable; reusable packages must not depend on UI, CLI, global flags, or external services. C++ integration uses the executable as an isolated subprocess; no C++ FFI layer is needed.

Single process, one active Test Run, multiple sessions, local reports. No Redis/Kafka/Kubernetes/external RDBMS, workflow engine, fleet controller, or distributed workers as V1 dependencies.

## Proposed source boundaries (not final filenames)

- `simulator/`: public, Go-importable config, client/session lifecycle, run API, structured result models.
- `internal/smpp/`: PDU encode/decode, session I/O, sequence correlation, timers.
- `internal/traffic/`: stream data generator, rate controller, window/distributor.
- `internal/metrics/`: low-overhead counters/histograms, DLR correlation view and capture limits.
- `internal/app/`: optional CLI, web server, headless integration adapter, report serialization.
- `cmd/smpp-studio/`: single entrypoint.
- `docs/`, `.codex/`: durable requirements and handoff.

Actual package names may be simplified further during implementation.

## Confirmed adapter invariants

D-032 and D-033 fix a consistent submit and DLR contract across CLI, Web UI, Go API and headless. **DLR text `id:` takes precedence over TLV `receipted_message_id`**, preserving conflicts in diagnostics; status `message_state` TLV remains first independently. D-034 fixes versioned headless outer input/output and exit codes, with no separate simulation engine. This is approved design, not implementation.

## D-038 shared public API invariants

The Go core must expose `New(Config)`, `Run(context.Context)`, `Stop`, `Close` with idempotent cleanup; its same validated schema-v1 effective config is used by CLI, embedded Web UI and noninteractive headless. Headless accepts named SMPP `connection.sessions` and never starts an HTTP listener. The process JSON report is redacted and versioned; no assertions are embedded, and no C++ native library or additional service is required. These are design commitments, not code.

## Critical invariants

- A session has safe serialized socket writes and an independent read loop; `deliver_sm_resp` is not starved behind large submit queues.
- Bound actual in-flight requests and memory consumption (including when response deadlines expire).
- Match `submit_sm_resp` by `sequence_number` rather than arrival order; do not confuse late responses with a later batch.
- Match DLRs by message ID and relevant context across sessions; avoid assuming every DLR arrives on the sender's TX connection.
- Never auto-retransmit a `submit_sm` with an uncertain acknowledgement, including after timeout, disconnect, or crash.
- DLR processing/ACK continues independently while send-side Adaptive Window is zero or a run is draining.
- Distinguish application traffic window=0 from active recovery probes and other connection health PDUs.
- Treat simulator generated faults in `deliver_sm_resp` separately from SMSC actual delivery outcome.

## Run lifecycle

Proposed phases: `CREATED -> CONNECTING -> RUNNING -> SUBMIT_DRAIN -> [DLR_DRAIN] -> COMPLETED` with terminal `STOPPED`, `INTERRUPTED`, `ERROR` as applicable. This enum and exact transition names remain design-level, not an approved schema.

- `RUNNING`: traffic generator feeds bounded dispatchers; rate controller applies a profile; session windows cap in-flight.
- `SUBMIT_DRAIN`: do not generate new messages; wait for outstanding submit responses until deadline, then mark unresolved/uncertain.
- `DLR_DRAIN` (only DLR-completion mode): observe receipts until expected DLRs resolve or configured wait ends; classify missing.
- `STOPPED`: for all-TX-lost/manual stop/safety stop; save available partial report and reason.
- `INTERRUPTED`: detected after a process crash/restart; never continue the old Test Run.
- `ERROR`: config/infrastructure/other terminal failure.

A single failed TX does not stop the run while another TX remains healthy. D-039 confirms that a recovered TX may rejoin as a new bind generation, accepting future messages only; previously ambiguous submits are never replayed. RX reconnection in-process is expected under RX-loss policies. In TRX setups, apply the TX-loss stop rule if all send-capable TRX sessions disappear.

## D-039 reconnect and run lifecycle

- Rebind named TX/RX/TRX sessions with configurable 1/2/4/8/16/30s-capped retry backoff and small jitter; avoid blind retries of permanent authentication errors.
- Recovered TXs rejoin as new session generations, with reset Adaptive controller and future work only. Initial absence of all TX binds is startup failure (exit 3); loss of all TX after Run starts is `all_tx_unavailable` (exit 4).
- RX-only loss follows Continue MT or Pause MT; receiver retry continues in-process, buffers stay bounded. The all-TX stop rule takes precedence.

## State and persistence

Use bounded in-memory running state and a minimal local report/snapshot writer. Do not require relational DB tables. Full capture may use sequential files; high-performance capture is optional/bounded. D-039 fixes 5s default snapshot flush, High Performance capture, 1,000 sampled PDU details and 512 MiB optional raw capture per Run. Capturing more than the quota truncates optional evidence but never counters. Reports remain local, with no automatic deletion; last successful flush time bounds known crash data loss. Store no plaintext credentials in logs/reports. Ensure test cancellation closes sockets, drains/terminates goroutines, and releases files without leaks.
