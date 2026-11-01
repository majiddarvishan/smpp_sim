# SMPP Test Studio — V1 Design Freeze

**Date:** 2026-11-01
**Status:** **FROZEN — DESIGN ONLY**
**Audited repository baseline:** `main` @ `4b1653293b53c3fc27159c958ce33a74dfbf9b17`

## Frozen owner-approved contract

All decisions **D-001–D-040** are `CONFIRMED`. The P0 owner-design checklist contains no unchecked decisions. The canonical documents are:

- [Decisions](DECISIONS.md) and [Product](PRODUCT_SPEC.md): scope, exclusions, SMPP v3.4, one Go engine and executable, no assertions/MO scenarios.
- [Protocol](PROTOCOL_DATA_FAULTS.md): D-032 message fields/encoding, D-033 **DLR text `id:` first** with TLV fallback (state independently TLV first), D-035/D-036 response faults, D-039 120s bounded DLR matching (default 1,000,000 active).
- [Traffic](TRAFFIC_CONTROL.md): six profiles, D-037 batch-based independent Adaptive Window, default 3s batch deadline separate from 5s per-PDU timeout, zero-window authorized probes and global caps.
- [Headless](HEADLESS_INTEGRATION.md): D-034/D-038 strict versioned JSON/YAML and JSON result, Go `New/Run/Stop/Close`, C++ subprocess, no built-in assertions or external service.
- [Architecture](ARCHITECTURE.md): D-016/D-017/D-039 TX and RX loss, new TX bind generations for future messages only, no uncertain-submit replay or restart resume; bounded local reports.
- [Acceptance](ACCEPTANCE.md): confirmed D-040 functional, safety, integration and benchmark targets.
- [Backlog](BACKLOG.md) and [Tasks](../.codex/TASKS.md): deferred scope and engineering remainder.

## Consistency audit results

1. GitHub content check: exactly **40 confirmed decision entries**, one each from D-001 through D-040.
2. No unchecked **P0 owner design tasks** after D-040. Remaining unchecked tasks are scoped engineering/implementation or future validation work, not silently treated as done.
3. D-037, D-038, D-039 and D-040 canonical sections exist; D-032/D-036/D-037 supersession notes reconciled.
4. Checked local Markdown document paths; no broken local `.md` document links in the audited set.
5. **No Go simulator, test run or benchmark** is claimed. Freeze is a documentary design result, not a functional acceptance result.

## Bounded engineering details, not new V1 owner decisions

- Optional Advanced TLV catalogs and vendor normalization adapters remain opt-in interoperability extensions. Only documented baseline protocol TLVs/fields are guaranteed; reject unknown unsupported input clearly.
- Implementation must choose conservative CSV invalid-row validation, explicit Advanced nested field types, non-Adaptive rate-controller numeric settings, SMPP congestion-status fixtures, bounded sequence tombstones/reconnect fencing and local snapshot mechanics while preserving all approved behavior. Never silently truncate, resend uncertain submits, normalize addresses or accept unbounded work.
- Any new or incompatible product behavior requires a new recorded owner decision; do not expand V1 by implication.

## D-040 release gates — **not executed, not PASS**

| Area | Future acceptance requirement |
| --- | --- |
| Protocol and operations | Automated SMPP v3.4, encoding, TON/NPI, multipart, DLR, faults, Adaptive recovery, connection and crash/reports tests |
| Language support | Direct Go API, strict JSON/YAML headless, C++ subprocess and machine-readable results with D-034 exit codes |
| Throughput | **>=50,000 actual physical `submit_sm` PDUs/s average over 5 minutes**; 8 TX/TRX benchmark sessions (not Basic default), DLR OFF, no optional raw capture, at least two reproducible runs |
| Reference environment | Linux x86_64, >=16 logical CPU, >=32 GiB RAM, loopback/isolated >=1 GbE, test peer independently verified >60k PDUs/s |
| Resources/evidence | Peak simulator RSS <8 GiB, no unexplained lost PDUs/safety stop, one-second TPS series, CPU/latency/error counts and environment evidence |
| Separate DLR workload | **5,000 submit PDUs/s for 3 minutes** with deterministic IDs/receipt and ACK correlation |
| Safety and distribution | Finite authorized destinations/probes, localhost-only default UI, environment secrets/redacted reports, local bounded storage, Linux and Windows amd64 smoke |

Performance targets are **unverified**. Never label a benchmark PASS without executed measurements, or replace the target by a lower number without explicit owner authorization.

## Gate to implementation

**FROZEN DESIGN DOES NOT AUTHORIZE IMPLEMENTATION.** D-023 remains binding. The owner must **separately and explicitly instruct** coding to begin. When permitted, implement in small tested increments, without mandatory GitHub Actions or a separate orchestration platform.
