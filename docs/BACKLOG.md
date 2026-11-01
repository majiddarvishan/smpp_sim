# Backlog, Exclusions and Open Decisions

Status: 2026-11-01. Do not treat candidate defaults as approved.

## Deferred beyond V1

- **TLS** for SMPP connections.
- **`query_sm`**, **`cancel_sm`**, **`replace_sm`**.
- Potential features beyond this list require explicit scope approval; avoid building a platform.

## Explicitly out of scope

- MO traffic generation, MO receipt test scenarios, MO injection tools.
- Built-in assertion rules/DSL or test-management service.
- Automatic resume/checkpoint replay after process restart/crash.
- Required external broker/database/distributed worker cluster.
- Native C++ SDK/FFI (use headless subprocess JSON results); separate orchestration server.
- SMSC-side `deliver_sm` generator/server implementation (this is a client simulator).

## Unsettled details to finalize before implementation

1. **SMPP parameters:** supported version/system type/address range, TON/NPI defaults, `enquire_link`/bind/unbind timeouts, `interface_version`, configured TLV coverage, `message_payload`, validity/schedule time format and encoding/multipart policy.
2. **Run config schema:** versioned JSON/YAML shape, CLI syntax, Web UI fields, headless result schema, exit-code contract with **no built-in assertions**.
3. **Traffic math:** precise pacing clock/limits, Adaptive increase/decrease/timeout thresholds, max/initial window defaults, zero-window transition and probe interval/success count, late response bookkeeping, per-session vs global controls.
4. **Connection lifecycle:** TX rejoin behavior when some TX remain; distinction between temporary RX outage, DLR drain, and all-TX-loss terminal stop; reconnect limits and TRX semantics.
5. **DLR handling:** receipt parsing forms, `message_id` normalization, delivery-status mapping, expected receipt eligibility, retention and expiry semantics.
6. **Fault injection:** exact allowed SMPP `command_status` values, percentage sampling and seed isolation, DLR classification, delay/no-response precedence and defaults, log/counter meanings.
7. **Data:** CSV schema and invalid-row rules, shuffle implementation for huge files, templates and random strategies, safety restrictions on test destinations.
8. **Storage:** local report files, flush intervals, crash-loss evidence, capture limits and retention; no mandatory DB.
9. **Acceptance:** measurable hardware/conditions for 50k TPS, sustained time, allowed p95/timeout resource use, end-to-end DLR load test workload, cross-platform C++ runner smoke test.
10. **Security and packaging:** secret injection and redaction, UI bind address/access defaults, authorized-target/real-number guardrails, reproducible builds.

## Short next design sequence

- Agree exact **SMPP connection options** and defaults, keeping advanced options collapsible in UI.
- Freeze minimal JSON/YAML run schema + JSON result contract for C++/Go integration.
- Freeze Adaptive Window and fault injection parameter semantics.
- Set objective V1 acceptance tests; only then begin coding in small, tested increments.

**Reminder:** no simulator implementation exists at the time of this documentation commit.
