# Decision Register

Recorded from product design discussion through **2026-11-01**. **Confirmed** = explicitly accepted or unambiguously required by the owner. Do not promote proposed defaults to confirmed design silently.

| ID | Status | Decision |
| --- | --- | --- |
| D-001 | CONFIRMED | Build a lightweight, maintainable **SMPP client simulator**, not a large operational platform. |
| D-002 | CONFIRMED | One Go core, single executable for CLI + embedded Web UI + headless; importable Go library; no mandatory external stack. |
| D-003 | CONFIRMED | Target **50,000 submitted `submit_sm` PDUs/s per instance**, to validate, not claim as achieved. |
| D-004 | CONFIRMED | Support Single SMS and Bulk tests, TX/RX/TRX binds, multi-session, simultaneous MT submit and DLR receive. |
| D-005 | CONFIRMED | **No MO scenarios.** MO receive/injection subsystem removed from scope. |
| D-006 | CONFIRMED | TLS, `query_sm`, `cancel_sm`, `replace_sm` postponed to backlog. |
| D-007 | CONFIRMED | Six V1 traffic methods: Constant, Ramp-up/down, Step Load, Burst/Spike, Unlimited, Adaptive Window. |
| D-008 | CONFIRMED | Adaptive Window is batch-based: send up to current window, wait bounded time for all responses, shrink on degraded rounds, grow when healthy, never over max, and may reach zero. |
| D-009 | CONFIRMED | At window=0, stop normal traffic and auto-recover via controlled periodic submit probes; no manual restart needed for recovery. |
| D-010 | CONFIRMED | Adaptive Windows independent per TX session, bounded also by global rate/safety controls. |
| D-011 | CONFIRMED | Traffic distribution configurable by Test Run: Equal, Adaptive and Weighted. |
| D-012 | CONFIRMED | Completion configurable: Submit-only or Submit+DLR wait/drain. |
| D-013 | CONFIRMED | Capture configurable: Full Detail vs High Performance; optional Sample Capture; accurate aggregate counts and DLR correlation independent of sampling. |
| D-014 | CONFIRMED | Process crash/restart: preserve available reports, mark Interrupted; **no automatic resume** and no blind resend of ambiguous requests. |
| D-015 | CONFIRMED | Dataset exhaustion policy configurable Stop at End vs Loop, with Sequential vs Shuffle order. |
| D-016 | CONFIRMED | Full DLR RX loss during otherwise active TX: per-run Continue MT or Pause MT; RX reconnect during live process. |
| D-017 | CONFIRMED | If one TX disconnects, continue through other healthy TX sessions. If **all TX** unavailable, **stop** Test Run. |
| D-018 | CONFIRMED | Automated integration for Go and other languages, **explicitly C++**. Go API plus headless JSON/YAML config and machine-readable JSON result; no CLI interaction or Web UI required in headless use. |
| D-019 | CONFIRMED | **No assertions within simulator.** External testing framework (including C++ Google Test etc.) owns pass/fail checks. |
| D-020 | CONFIRMED | Configurable percentage-based fault injection for incoming `deliver_sm` response, with DLR-specific override, status code, delay and no-response options. |
| D-021 | CONFIRMED | Favor limited dependencies and single active Test Run per instance; avoid complex distributed recovery/orchestration systems. |
| D-022 | CONFIRMED | **V1 targets SMPP v3.4 only.** Other protocol versions are not included in V1. |
| D-023 | CONFIRMED | **Design-only phase continues. Do not start implementation until explicitly authorized by the owner.** |
| D-024 | CONFIRMED | V1 connection defaults: **SMPP 3.4**, **TCP**, **TRX**, **1 connection**, **connect timeout 5s**, **bind timeout 10s**, **submit response timeout 5s**, **enquire_link interval 30s**, **enquire_link response timeout 10s**, **empty system_type**, **empty address_range**; all timeout values configurable. |
| D-025 | CONFIRMED | Automatic `enquire_link` may be disabled for tests; exact config key/syntax remains to be decided. |
| D-026 | CONFIRMED | V1 message encoding defaults to **Auto**: choose **GSM 03.38/GSM7** when the complete text is representable in its default/extension alphabet, otherwise **UCS2** for supported characters (including Persian). **Manual** choice of GSM7, UCS2, and binary plus explicit advanced `data_coding` configuration is supported for protocol tests. Exact wire encoding/interoperability and unsupported Unicode policy need design confirmation. |
| D-027 | CONFIRMED | **Multipart SMS Auto + Manual** for V1: default automatic splitting of long messages using **UDH**; Advanced can explicitly choose **UDH**, **SAR TLVs**, or **no automatic splitting**. CLI, Web UI, headless and Go API must share the selected behavior. Logical SMS count and actual `submit_sm` PDU/segment count are reported separately. Exact size limits, failure/oversize handling in no-split mode, UDH reference strategy and SAR-specific wire validation are pending design. |
| D-028 | CONFIRMED | **Auto + Manual TON/NPI** in V1. Default Basic behavior automatically derives **source** and **destination** TON/NPI independently when the address format gives an unambiguous indication (e.g. alphanumeric sender `TON=5, NPI=0`; explicitly international address `TON=1, NPI=1`). Ambiguous inputs use a conservative default, whose exact mapping is still open. Advanced and programmatic inputs can override TON and NPI independently. Selecting TON/NPI **must never normalize or change the actual source/destination address string**. |

## Interpretation notes

- SMPP DLR is ordinarily carried inside `deliver_sm`. D-020 does not add an independent `delivery_report` PDU or MO feature.
- Adaptive Window zero does **not** mean no traffic at all: small explicit probes are allowed and reported separately.
- The rule to stop on all-TX loss overrides recovery of sending sessions for that run. RX in-process reconnect remains valid where TX is still healthy.
- Previous examples of Adaptive Window sizes/timers, error distributions and exit codes remain illustrative. Connection defaults are explicitly approved in D-024. The `submit_sm_resp` timeout and Adaptive Window per-batch deadline are different timers.
- A successful `submit_sm_resp` is not proof of final SMS delivery.
- D-028 selects **TON/NPI metadata only**, not rewriting addresses. Exact precedence for numeric-only or ambiguous formats, a leading `+`, and prefixes is still an open design detail; do not silently infer international TON from all digit-only strings.
- D-027 specifies **default automatic UDH fragmentation** as a product choice, not universal SMSC compatibility. **No automatic splitting** is an explicit test mode, not permission to truncate or silently alter long payloads.
- D-026 specifies the **product default selection policy**, not an SMPP-standard default: `data_coding=0x00` signals the SMSC default alphabet and does not guarantee the SMSC's wire GSM7 packing assumptions. `0x08` is the usual UCS2 indicator. Auto must not silently corrupt characters; exact implementation semantics remain pending.

## Future entries

Add dated records when the owner accepts precise defaults/config schemas/engineering trade-offs. Never rewrite the decision register to hide superseded decisions; append an explicit superseding entry.
