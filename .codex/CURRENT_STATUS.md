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
- **TON/NPI Auto + Manual** approved in D-028: independent source/destination classification and overrides; no implicit address modification. Approved D-029: digit-only without `+` uses `0/0`, leading-plus digits use `1/1`, clear alphanumeric uses `5/0`; malformed address validation is still open.
- DLR requests default OFF (`registered_delivery=0x00`) with per-run Basic toggle and Advanced full-field override, confirmed in D-030. Basic ON is `0x01` (D-031); Advanced `0x02` and explicit `0x03` vendor test (reserved in SMPP 3.4) are approved.
- **D-032 CONFIRMED:** normal `submit_sm`, unchanged Basic ASCII addresses, GSM7 unpacked/UCS2 BMP/binary wire handling, UDH 16-bit default/8-bit and SAR alternatives, no-split and explicit payload.
- **D-033 CONFIRMED:** DLR message ID from **text `id:` first**, TLV fallback; on conflict text wins with diagnostic, while status independently prefers `message_state` TLV; per-segment correlation and 60s drain.
- **D-034 CONFIRMED:** v1 versioned headless JSON/YAML and JSON result, finite run, C++ subprocess integration, no assertions, execution exit codes `0/2/3/4/5/6`.
- **Design only**: product owner explicitly requested no implementation yet.
- Key product decisions: `docs/DECISIONS.md`.

## Next discussion
D-032–D-034 establish the baseline protocol and headless outer contract. Next design tasks: precise Adaptive Window controls, fault-injection probabilities/action timing, nested headless keys/types, DLR correlation retention and measured acceptance workloads. **No implementation authorized**.

## Safety/quality notes
Do not claim runnable code, successful tests, acceptance, or implementation progress until actually executed and verified. Do not add GitHub Actions by default.
