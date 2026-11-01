# Backlog, Exclusions and Open Decisions

Status: 2026-11-01. Do not treat candidate defaults as approved.

## Deferred beyond V1

- SMPP protocol versions other than **3.4** (only if a real need arises).
- **TLS** for SMPP connections.
- **`query_sm`**, **`cancel_sm`**, **`replace_sm`**.
- Potential features beyond this list require explicit scope approval; avoid building a platform.

## Explicitly out of scope

- MO traffic generation, MO receipt test scenarios, MO injection tools.
- Built-in assertion rules/DSL or test-management service.
- Automatic resume/checkpoint replay after process restart/crash.
- Required external broker/database/distributed worker cluster.
- Native C++ SDK/FFI (use headless subprocess JSON results); separate orchestration server.
- SMSC-side `deliver_sm` generator/server implementation (this is a client simulator).

## Unsettled details to finalize before implementation

1. **SMPP parameters:** SMPP 3.4 only; connection defaults (D-024), optional automatic enquire_link (D-025), Auto + Manual encoding (D-026), multipart mode selection (D-027), and Auto + Manual TON/NPI **selection policy** (D-028) are approved. Open: malformed-address validation (the normal TON/NPI mapping is fixed in D-029, and addresses are never altered), unbind timeout, version field encoding, TLV coverage, `message_payload`, validity/schedule format, exact GSM7 packing/byte handling, non-BMP Unicode policy, UCS2/manual mismatch validation, UDH/SAR reference/fragmentation details, no-split oversize behavior and multipart DLR mapping.
2. **Run config schema:** versioned JSON/YAML shape, CLI syntax, Web UI fields, headless result schema, exit-code contract with **no built-in assertions**.
3. **Traffic math:** precise pacing clock/limits, Adaptive increase/decrease/timeout thresholds, max/initial window defaults, zero-window transition and probe interval/success count, late response bookkeeping, per-session vs global controls.
4. **Connection lifecycle:** TX rejoin behavior when some TX remain; distinction between temporary RX outage, DLR drain, and all-TX-loss terminal stop; reconnect limits and TRX semantics.
5. **DLR handling:** DLR request OFF by default with Basic toggle and Advanced override approved (D-030); Basic ON `0x01`, Advanced `0x02`, and explicit vendor-specific `0x03` are decided (D-031); still specify how to classify missing receipts for SMSCs that reject/ignore reserved `0x03`;   receipt parsing forms, `message_id` normalization, delivery-status mapping, expected receipt eligibility, retention and expiry semantics.
6. **Fault injection:** exact allowed SMPP `command_status` values, percentage sampling and seed isolation, DLR classification, delay/no-response precedence and defaults, log/counter meanings.
7. **Data:** CSV schema and invalid-row rules, shuffle implementation for huge files, templates and random strategies, safety restrictions on test destinations.
8. **Storage:** local report files, flush intervals, crash-loss evidence, capture limits and retention; no mandatory DB.
9. **Acceptance:** measurable hardware/conditions for 50k TPS, sustained time, allowed p95/timeout resource use, end-to-end DLR load test workload, cross-platform C++ runner smoke test.
10. **Security and packaging:** secret injection and redaction, UI bind address/access defaults, authorized-target/real-number guardrails, reproducible builds.

## Short next design sequence

- Finalize remaining **submit_sm/DLR protocol details**: malformed-address validation (Auto+Manual TON/NPI and common defaults already approved), GSM7/UCS2 wire behavior, non-BMP Unicode, multipart UDH/SAR framing and DLR mapping.
- Freeze minimal JSON/YAML run schema + JSON result contract for C++/Go integration.
- Freeze Adaptive Window and fault injection parameter semantics.
- Set objective V1 acceptance tests; only then begin coding in small, tested increments.

**Reminder:** no simulator implementation exists at the time of this documentation commit.

## Candidate closure package — 2026-11-01 (PENDING OWNER APPROVAL)

**Status: proposed, NOT CONFIRMED.** This section saves a compact decision batch for owner review; it does **not** create D-032+ or close P0 checkboxes. Existing D-001 through D-031 remain unchanged. No simulator code is authorized.

### A. Remaining `submit_sm` defaults (candidate D-032)

- Normal PDU defaults: `service_type=""`, `esm_class=0x00` unless multipart UDH sets UDHI (`0x40`), `protocol_id=0`, `priority_flag=0`, `schedule_delivery_time=""` (immediate), `validity_period=""` (SMSC policy), `replace_if_present_flag=0`, `sm_default_msg_id=0`, no extra TLVs. `interface_version=0x34` for bind. Existing `registered_delivery` D-030/D-031 and source/destination Auto TON/NPI D-028/D-029 take precedence.
- Basic address validation: destination required, source may be empty for SMSC default; at most 20 address octets before C-octet terminator for each; reject NUL, non-ASCII, whitespace or malformed leading `+` forms; accept exact ASCII alphanumeric, digit-only and `+digits` input **without changing a byte**. Advanced may intentionally send printable noncanonical ASCII addresses with explicit TON/NPI overrides, within SMPP field length limits; validation cannot silently normalize addresses.
- Auto GSM 03.38 selection uses the entire expanded message and escape-table septet counts. Proposed baseline SMPP GSM7 wire representation is **unpacked GSM alphabet octets** with `data_coding=0x00` (SMSC default alphabet); packed mode, if supported, is a deliberate Advanced interop choice, never an undocumented automatic switch. UCS2 is strict BMP text encoded as UTF-16BE with `data_coding=0x08`; reject non-BMP/surrogate input by default with an actionable error rather than misleadingly label it UCS2. Binary is exact bytes, proposed `data_coding=0x04`. A deliberate Advanced raw-DCS override may produce a compatibility warning but must never rewrite message bytes.
- Segmentation budgets based on GSM mobile user-data capacity, distinct from SMPP `short_message` field limit: single GSM7 160 septets / UCS2 70 BMP code units / binary 140 octets. Proposed Auto UDH reference is **16-bit**: `06 08 04 ref_hi ref_lo total part`, `esm_class` UDHI set; multipart GSM7 152 septets, UCS2 66 code units, binary 133 octets per part. Optional 8-bit UDH compatibility mode `05 00 03 ref total part` uses 153/67/134 respectively. Never split inside a GSM escape pair or UCS2 code unit; maximum 255 parts. References are constant across a logical SMS and must minimize reuse for the same destination; expose collision risk at extreme traffic rates, do not claim impossibility.
- Manual SAR TLVs use `sar_msg_ref_num` (16-bit), `sar_total_segments` and `sar_segment_seqnum` (1..255) on each segment, with no UDH/UDHI. Do not combine SAR and UDH. Use SMS user-data budgets, not the 254-octet SMPP field maximum, for network-targeted segments.
- "No automatic splitting" sends one unmodified `short_message` only when encoded length <=254 octets; warn when it exceeds the usual single-SMS network capacity. Greater than 254 octets => pre-send validation error unless user explicitly selects Advanced `message_payload`; that TLV uses `sm_length=0` and empty `short_message`, is never silently selected and may be SMSC-dependent. Automatic modes must not silently use `message_payload`.

### B. DLR matching and accounting (candidate D-033)

- Receive DLR via `deliver_sm` on RX/TRX independent of submit session. Classify from `esm_class` receipt indicator and/or coherent standard DLR TLVs; parse `receipted_message_id`/`message_state` TLVs first; support SMPP Appendix B `id:... stat:...` text fallback when TLVs absent. Flag conflicting TLV/text values; retain raw evidence in bounded capture. Non-DLR `deliver_sm` is ACKed and counted as ignored/unclassified input, **not** treated as an MO testing scenario.
- Default ID correlation is **exact opaque, case-sensitive matching** of the decoded C-octet message ID, across all RX/TX sessions. Do not guess decimal-to-hex conversion, trim significant zeros or change case. A per-target explicit compatibility option may specify a known canonicalization rule. Duplicate IDs producing more than one candidate are marked `ambiguous`, never arbitrarily assigned.
- One accepted `submit_sm_resp` with usable `message_id` creates one segment-level DLR correlator. Multipart logical outcome is derived from segment evidence; don't assume one receipt per logical SMS, and don't call an SMS delivered if some segments are rejected/unresolved. Track `accepted_without_id` separately.
- Receipt eligibility: `registered_delivery` low bits `0x00` => none guaranteed; `0x01` => one final receipt **expected per accepted, identifiable submitted PDU**; `0x02` => receipt conditional on final failure (absence does not prove missing); `0x03` => reserved SMPP 3.4 vendor test (no guaranteed receipt). For raw overrides, account for nonstandard bits and compatibility warnings instead of assuming universal final DLR. Submit + DLR drain waits only for guaranteed expected receipts; never invent missing DLRs for conditional/vendor cases.
- Terminal states: delivered only for `DELIVERED`/`DELIVRD`; failed for `EXPIRED`, `DELETED`, `UNDELIVERABLE`, `REJECTED` and matching text; `ENROUTE` intermediate; `ACCEPTED` and `UNKNOWN` tracked explicitly, not auto-counted as delivered. Preserve raw SMSC state and diagnostic error. Report `unmatched`, `duplicate`, `ambiguous`, `late`, `missing`, `unsolicited`; negative or uncertain submit outcomes are not presumed accepted.
- Candidate default DLR-drain deadline **60s**, configurable and independent of 5s submit-response timeout. The timeout is an *observation window*, not proof of delivery failure; preserve late/unresolved classification and never retry ambiguous submits.

### C. Headless runner contract (candidate D-034)

- One existing executable: `smpp-studio run --headless --config <path.json|path.yaml> --result <path.json>`. No UI listener, interactive prompts, service, native C++ SDK, separate Go engine, or built-in assertions.
- Use `schema_version: 1` for both input and result. Require explicit target host, port, system ID, credential from environment (e.g. `password_env`), source/destination for fixed messages, and **finite termination** (count, duration, or finite Stop-at-End dataset). No implicit unlimited sends. Reject unknown config keys and impossible combinations with structured diagnostics; JSON/YAML must resolve to identical effective settings.
- Candidate config top-level groups: `schema_version`, `connection`, `message`, `traffic`, `completion`, optional `capture`, `faults`, `seed`. Reuse the Go engine config; the basic run defaults inherit D-024 through D-031. Provide `--result` as separate output location, not two competing settings.
- Result JSON always attempted (including failed start), written atomically where feasible. Top-level: `schema_version`, `run_id`, `run_status`, `termination_reason`, `started_at`, `finished_at`, `effective_config` **redacted**, `counters`, `per_session`, `dlr`, `diagnostics`. Stable numeric counters include logical SMS attempted, physical PDUs sent, submit OK/rejected/timeout/uncertain, segment outcomes and DLR receipt categories. Bounded detail samples only; total counters remain accurate when sampling is off. Never emit credential values.
- Proposed process exit codes express execution health **not test assertions**: `0` = run finished as requested even with SMSC rejections/missing DLR; `2` = configuration/input error; `3` = startup/connect/bind failure; `4` = runtime could not complete, including all-TX loss or safety stop; `5` = cancelled/deadline; `6` = internal or result-write error. The result's termination reason is more detailed than an exit code. Caller (C++ Google Test/Catch2, Go, Python, Java) decides test pass/fail from the JSON.
- Deterministic input generation uses explicit seed; timing/network scheduling and remote SMSC outcomes are not promised bit-for-bit deterministic. Preserve CLI/Go/headless semantic parity.

**Owner review needed:** approve/revise/reject the three candidate decisions as a batch. Only after approval append D-032/D-033/D-034 to `docs/DECISIONS.md`, propagate into PRODUCT_SPEC/HEADLESS/PROTOCOL/ACCEPTANCE and check applicable P0 boxes. Keep fault-injection percentages, adaptive-window tuning, persistence, and performance acceptance as separate remaining tasks.
