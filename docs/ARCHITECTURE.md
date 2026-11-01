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

A single failed TX does not stop the run while another TX remains healthy. Whether previously lost TX sessions may rejoin the active pool is **not yet specified**; no replay of old submits is allowed. RX reconnection in-process is expected under RX-loss policies. In TRX setups, apply the TX-loss stop rule if all send-capable TRX sessions disappear.

## State and persistence

Use bounded in-memory running state and a minimal local report/snapshot writer. Do not require relational DB tables. Full capture may use sequential files; high-performance capture is optional/bounded. Document flushing interval and crash data-loss window before implementation. Store no plaintext credentials in logs/reports. Ensure test cancellation closes sockets, drains/terminates goroutines, and releases files without leaks.
