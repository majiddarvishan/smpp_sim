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

D-035 fixes the shared three-outcome `deliver_sm_resp` selection model (success, configured error, no response), full DLR override, percentage remainder, optional ACK delay and seeded selection with accurate counters. The detailed timing, allowed status values and scheduling capacity remain unapproved. Do not implement until the owner separately authorizes code.

## D-036 fixed delay and bounded ACK policy

Confirmed: ACK Fixed delay defaults to 0ms (disabled) and uses a separate post-outcome percentage; default maximum delay is 60s, configurable. Initial nonzero negative-status allowlist: `ESME_RSYSERR`, `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`. Maximum delayed pending ACKs defaults to 10,000 per run; reaching it requires controlled `resource_limit` termination, not silent policy changes. No Response schedules no timer or ACK. D-036 does not authorize implementation.

## Development discipline

Prefer changes scoped to agreed features; tests and docs updated together. Clearly separate accepted decisions from proposals. Never invent performance PASS/results. Keep config credentials out of VCS and logs. Do not overwrite prior design decisions silently. Favor local repeatable tests.
