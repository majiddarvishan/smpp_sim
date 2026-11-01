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
- **Auto + Manual encoding** selection approved in D-026. GSM7/UCS2 wire mapping and unsupported Unicode policy are still open.
- **Multipart Auto + Manual** choice approved in D-027 (default automatic UDH; Advanced UDH, SAR TLVs, or no splitting). Precise framing and oversize/fragment-specific DLR rules remain open.
- **Design only**: product owner explicitly requested no implementation yet.
- Key product decisions: `docs/DECISIONS.md`.

## Next discussion
Finalize remaining `submit_sm` and DLR field semantics (starting with TON/NPI, exact UDH/SAR framing and no-split oversize behavior); then headless contract and fault-injection tuning. **No implementation authorized**.

## Safety/quality notes
Do not claim runnable code, successful tests, acceptance, or implementation progress until actually executed and verified. Do not add GitHub Actions by default.
