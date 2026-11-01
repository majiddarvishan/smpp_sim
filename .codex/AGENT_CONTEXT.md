# Agent Context — smpp_sim

Read this file before work. The product is **SMPP Test Studio**, a standalone SMPP **client** simulator. This project is in **design phase**; do not start implementing without a user request.

## Source of truth

- `docs/DECISIONS.md`: confirmed product choices.
- `docs/PRODUCT_SPEC.md`: V1 scope.
- `docs/ARCHITECTURE.md`, `docs/TRAFFIC_CONTROL.md`, `docs/PROTOCOL_DATA_FAULTS.md`, `docs/HEADLESS_INTEGRATION.md`: design constraints.
- `docs/BACKLOG.md`: explicit deferrals and unresolved decisions.
- `.codex/TASKS.md`, `.codex/CURRENT_STATUS.md`: what is next and what actually happened.

## Non-negotiables

- Maintainable, small Go engine; CLI/Web UI/headless adapters share it. Single binary for manual use, **no required third-party infrastructure**.
- C++ integration is essential; headless process config in JSON/YAML, JSON result; Go direct import API.
- External caller owns assertions. Do not implement assertions inside simulator.
- Goal 50k `submit_sm` PDUs/s per instance is an **unverified benchmark goal**.
- Six traffic profiles including batch Adaptive Window with zero-window automatic probe recovery; independent TX windows; Equal/Adaptive/Weighted routing.
- Concurrent DLR receive and configurable ACK fault injection. **No MO scenarios.**
- All TX unavailable => STOP run; one TX lost => continue. RX-only full outage => policy selectable continue/pause with RX reconnect while running.
- Crash/restart => Interrupted, never auto-resume.
- TLS/query/cancel/replace are backlog.
- No mandatory CI workflow (especially GitHub Actions), large orchestrator, or generalized production SMSC product.

## Latest confirmed decisions (2026-11-01)

- **D-032:** strict preserved Basic addresses, `submit_sm` defaults, GSM7 unpacked/UCS2 BMP/binary wire formats, 16-bit default and 8-bit optional UDH, SAR/no-split/explicit payload handling.
- **D-033:** DLR message ID **text `id:` first**, then `receipted_message_id` TLV fallback; text ID wins on conflicting values with diagnostics. Delivery state is independently `message_state` TLV-first, then text `stat:` fallback. Per-segment matching, 60s drain default, conditional/vendor receipt rules.
- **D-034:** single versioned headless runner with JSON/YAML input, JSON output, C++ subprocess support, and exit codes `0/2/3/4/5/6`; no simulator assertions.
- **D-023:** No implementation authorization. Follow canonical protocol and headless specs, and retain only unresolved detail/tuning in backlog.

## D-035 approved response policy

D-035 fixes the shared three-outcome `deliver_sm_resp` selection model (success, configured error, no response), full DLR override, percentage remainder, optional ACK delay and seeded selection with accurate counters. D-036 subsequently confirmed fixed delay defaults, the initial four statuses and the 10,000 delayed-response scheduling cap; do not treat them as open. Do not implement until the owner separately authorizes code.

## D-036 fixed delay and bounded ACK policy

Confirmed: ACK Fixed delay defaults to 0ms (disabled) and uses a separate post-outcome percentage; default maximum delay is 60s, configurable. Initial nonzero negative-status allowlist: `ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`. Maximum delayed pending ACKs defaults to 10,000 per run; reaching it requires controlled `resource_limit` termination, not silent policy changes. No Response schedules no timer or ACK. D-036 does not authorize implementation.

## D-037 approved Adaptive Window tuning

Independent per-TX/TRX batch windows: initial 32, max 1,024, batch deadline 3s versus per-submit response timeout 5s, healthy p95<=1s, step increase max(1,floor(window/8)), degrade half, three severe consecutive rounds to zero, 5s single-flight authorized probes and two successive OK responses reopen at window=1. Shared maximum 8,192 pending PDUs including probes and configurable 10m finite Adaptive run deadline. Correct bounded late response tracking; no blind resend; all TX loss stops the run, RX-only loss follows D-016. See `docs/TRAFFIC_CONTROL.md`. No implementation is authorized.

## D-038 headless V1 contract

Confirmed: one schema-v1 JSON/YAML outer config, strict `connection.sessions` plus `message`/`traffic`/`completion`, optional `capture`/`faults`/`seed`; finite runs, env-only secrets, no legacy `test` group, strict unknown-key rejection. JSON result has D-034 status/reason, redacted config and per-session counters. Go public lifecycle is `New`, `Run(context.Context)`, `Stop`, idempotent `Close`; C++ uses same executable subprocess, no SDK or embedded test assertions. See `docs/HEADLESS_INTEGRATION.md`. No implementation approved.

## D-039 connection and reporting policy

Confirmed: controlled reconnect backoff, returned TX as a new bind generation, DLR 120s TTL/1,000,000-cap, local High Performance reports, 5s snapshots and no automatic report deletion. Never replay uncertain messages or restart interrupted runs.

## D-040 approved acceptance and freeze gate

D-040 confirms automated SMPP/DLR/Fault/Adaptive/recovery/Headless C++ and Go acceptance, 50k *actual physical* submit PDUs/s sustained five minutes on a peer-qualified reference host, separate 5k submit PDUs/s DLR scenario for three minutes, 8 GiB reference RSS target, two reproducible performance runs, finite authorized tests, localhost UI, secret redaction and Linux/Windows amd64 smoke. **These are targets only: none have been executed.** Design freeze requires consistent documents; it never authorizes implementation without separate owner request per D-023.

## V1 frozen design and implementation authorization

As of 2026-11-01, owner decisions D-001–D-040 are CONFIRMED, documentary consistency audit passed, and V1 design is **FROZEN**. See `docs/V1_DESIGN_FREEZE.md`. **Do not write simulator code, create a Go module, execute benchmarks or claim test PASS without separate owner authorization to implement** (D-023). Implementation-oriented engineering details remain unchecked, not owner-level unresolved product decisions. D-040's 50k physical submit PDU/s for five minutes and separate 5k with-DLR PDU/s for three minutes are future measured release gates, not observed performance.

## Development discipline

Prefer changes scoped to agreed features; tests and docs updated together. Clearly separate accepted decisions from proposals. Never invent performance PASS/results. Keep config credentials out of VCS and logs. Do not overwrite prior design decisions silently. Favor local repeatable tests.
