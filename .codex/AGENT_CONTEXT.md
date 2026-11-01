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

## Development discipline

Prefer changes scoped to agreed features; tests and docs updated together. Clearly separate accepted decisions from proposals. Never invent performance PASS/results. Keep config credentials out of VCS and logs. Do not overwrite prior design decisions silently. Favor local repeatable tests.
