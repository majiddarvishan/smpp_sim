# Current Status — smpp_sim

Date: **2026-11-01**
Phase: **Specification / architecture; docs only**.

## Facts
- Repository initialized with a README and persistent product/technical decision documentation.
- No Go module, CLI/Web UI, SMPP engine, headless runner, tests or benchmarks have been implemented.
- Performance target of 50k `submit_sm` PDUs/s is a future benchmark objective, not demonstrated.
- Product owner asked for light maintainable design, automated integration including C++, and no built-in assertions.
- SMPP **3.4 only** is confirmed for V1.
- Approved connection defaults and the ability to disable automatic enquire_link are recorded in D-024 and D-025.
- **Design only**: product owner explicitly requested no implementation yet.
- Key product decisions: `docs/DECISIONS.md`.

## Next discussion
Finalize `submit_sm` and DLR field semantics/defaults (starting with TON/NPI and encoding/multipart), then headless contract and fault-injection tuning. Still **design only**.

## Safety/quality notes
Do not claim runnable code, successful tests, acceptance, or implementation progress until actually executed and verified. Do not add GitHub Actions by default.
