# SMPP Test Studio

A lightweight Go SMPP client simulator with CLI, embedded Web UI, and headless integration testing.

**Status:** Requirements and design only. Full requirements live under `docs/` and decisions are tracked in `docs/DECISIONS.md`.

**Performance target:** 50,000 `submit_sm` PDUs/s per instance; a benchmark target, not a measured guarantee.

**Scope:** MT submission, `submit_sm_resp`, inbound DLR via `deliver_sm` with configurable ACK fault injection; no MO scenarios. TLS, `query_sm`, `cancel_sm`, `replace_sm` are backlog.

Architecture goal: shared Go library/engine, single binary, no required external operational services. C++/Python/Java test suites invoke the JSON/YAML-driven headless runner and inspect JSON results; assertions stay in the calling test framework.
