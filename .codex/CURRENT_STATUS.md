# Current Status — smpp_sim

Date: **2026-11-01**
Phase: **V1 DESIGN FROZEN — D-001–D-040 confirmed and consistency-audited; docs only, no implementation authorization**.

## Facts
- Repository initialized with a README and persistent product/technical decision documentation.
- No Go module, CLI/Web UI, SMPP engine, headless runner, tests or benchmarks have been implemented.
- Performance target of 50k `submit_sm` PDUs/s is a future benchmark objective, not demonstrated.
- Product owner asked for light maintainable design, automated integration including C++, and no built-in assertions.
- SMPP **3.4 only** is confirmed for V1.
- Approved connection defaults and the ability to disable automatic enquire_link are recorded in D-024 and D-025.
- **Auto + Manual encoding** is approved in D-026; D-032 confirms the GSM7 unpacked/DCS 0x00, UCS2 BMP UTF-16BE/DCS 0x08, binary DCS 0x04 and non-BMP rejection baseline. Vendor interoperability fixtures remain engineering work.
- **Multipart Auto + Manual** choice approved in D-027 (default automatic UDH; Advanced UDH, SAR TLVs, or no splitting). D-032/D-033 now fix baseline framing, oversize handling and per-segment DLR rules; only vendor-specific interoperability remains to validate.
- **TON/NPI Auto + Manual** approved in D-028/D-029: digit-only without `+` is 0/0, plus-prefixed digits 1/1, alphanumeric 5/0, with no address rewrite. D-032 confirms strict Basic ASCII address validation; advanced interoperability remains engineering work.
- DLR requests default OFF (`registered_delivery=0x00`) with per-run Basic toggle and Advanced full-field override, confirmed in D-030. Basic ON is `0x01` (D-031); Advanced `0x02` and explicit `0x03` vendor test (reserved in SMPP 3.4) are approved.
- **D-032 CONFIRMED:** normal `submit_sm`, unchanged Basic ASCII addresses, GSM7 unpacked/UCS2 BMP/binary wire handling, UDH 16-bit default/8-bit and SAR alternatives, no-split and explicit payload.
- **D-033 CONFIRMED:** DLR message ID from **text `id:` first**, TLV fallback; on conflict text wins with diagnostic, while status independently prefers `message_state` TLV; per-segment correlation and 60s drain.
- **D-034 CONFIRMED:** v1 versioned headless JSON/YAML and JSON result, finite run, C++ subprocess integration, no assertions, execution exit codes `0/2/3/4/5/6`.
- **D-035 CONFIRMED:** one of three ACK response outcomes per inbound `deliver_sm`, complete DLR-policy override, error/no-response percentage budgets and success remainder, optional ACK delay and seeded selection.
- **D-036 CONFIRMED:** Fixed delay 0ms default, optional independent ACK-delay percentage after outcome selection, configurable 60s default maximum; four initial error codes; default maximum 10,000 pending delayed ACKs and controlled `resource_limit` termination when exhausted. No Response creates no timer.
- **D-037 CONFIRMED:** independent Adaptive Window defaults (initial 32/max 1,024; 3s batch decision versus 5s submit timeout; p95 1s; +12.5% rounded down, min 1; half-shrink; three consecutive severe batches to window zero; authorized 5s single-flight probes; two successes recover at window 1; 8,192 run-wide pending cap, configurable 10m run deadline). No blind resends; late-response accounting independent of capture.
- **D-038 CONFIRMED:** strict nested headless schema v1 using named `connection.sessions`, existing top-level groups, fixed/dataset/binary input, finite work and one effective config, UTC structured redacted JSON results, unchanged D-034 exit codes, four-operation Go lifecycle, C++ subprocess parity and no assertions.
- **D-040 CONFIRMED:** owner approved V1 functional and cross-language acceptance gates, reference 50,000 physical submit PDUs/s for 5 minutes, separate 5,000 submit PDU/s for 3 minutes with DLR, and safety/packaging checks. These benchmarks have NOT run and no functional PASS is claimed.
- **Design only**: product owner explicitly requested no implementation yet.
- Key product decisions: `docs/DECISIONS.md`.

- **D-039 CONFIRMED:** controlled reconnection and new TX generations, bounded 120s DLR correlation/1,000,000 mappings, local High Performance reports with 1,000 samples, 512 MiB raw capture and 5s snapshot flush. No automatic artifact deletion, replay or resume.

## Design freeze record

- On 2026-11-01, repository design consistency audit verified exactly one CONFIRMED entry per D-001–D-040, no unchecked P0 owner design decision, valid local documentation links and aligned D-037–D-040 canonical sections. See `docs/V1_DESIGN_FREEZE.md`.
- **Design FROZEN**, but no Go simulator/CLI/Web UI/headless runner has been built and **no test or benchmark has been executed**; 50k submit PDU/s remains an unverified objective.
- D-023 still explicitly prohibits implementation until the owner gives a separate request. Engineering detail tasks and P1 implementation checks remain unchecked.

## Next discussion
D-001–D-040 and the documentary consistency audit are complete: V1 design is **FROZEN**. The next step is **not automatic coding**; wait for separate explicit owner authorization, then implement in small tested increments and measure D-040 acceptance. **No implementation authorized; no tests or benchmarks executed.**

## Safety/quality notes
Do not claim runnable code, successful tests, acceptance, or implementation progress until actually executed and verified. Do not add GitHub Actions by default.
