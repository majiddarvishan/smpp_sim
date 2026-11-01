# SMPP Protocol, Datasets, DLR and Fault Injection

## Protocol scope

V1 supports plain TCP SMPP client binds TX/RX/TRX, sends `submit_sm` and consumes `submit_sm_resp`. It receives delivery receipts as inbound `deliver_sm` and sends matching `deliver_sm_resp` (same request `sequence_number`). **Delivery report is not an independent SMPP command**. Do not implement MO test flows.

Required submit/message fields include source/destination TON/NPI, addresses, `data_coding`, `esm_class`, `registered_delivery`, message body or payload and optional TLVs. Include GSM7/UCS2/binary and multipart UDH/SAR paths as supported by the target SMPP profile. D-032/D-033 now fix baseline encoding, validity, multipart framing, no-split and DLR correlation. Advanced TLV coverage and vendor interoperability fixtures remain open.

## Confirmed `submit_sm` defaults (D-032)

- Normal PDU defaults: `service_type=""`, `esm_class=0x00` unless multipart UDH sets UDHI (`0x40`), `protocol_id=0`, `priority_flag=0`, `schedule_delivery_time=""` (immediate), `validity_period=""` (SMSC policy), `replace_if_present_flag=0`, `sm_default_msg_id=0`, no extra TLVs. `interface_version=0x34` for bind. Existing `registered_delivery` D-030/D-031 and source/destination Auto TON/NPI D-028/D-029 take precedence.
- Basic address validation: destination required, source may be empty for SMSC default; at most 20 address octets before C-octet terminator for each; reject NUL, non-ASCII, whitespace or malformed leading `+` forms; accept exact ASCII alphanumeric, digit-only and `+digits` input **without changing a byte**. Advanced may intentionally send printable noncanonical ASCII addresses with explicit TON/NPI overrides, within SMPP field length limits; validation cannot silently normalize addresses.
- Auto GSM 03.38 selection uses the entire expanded message and escape-table septet counts. Default SMPP GSM7 wire representation is **unpacked GSM alphabet octets** with `data_coding=0x00` (SMSC default alphabet); packed mode, if supported, is a deliberate Advanced interop choice, never an undocumented automatic switch. UCS2 is strict BMP text encoded as UTF-16BE with `data_coding=0x08`; reject non-BMP/surrogate input by default with an actionable error rather than misleadingly label it UCS2. Binary is exact bytes, default `data_coding=0x04`. A deliberate Advanced raw-DCS override may produce a compatibility warning but must never rewrite message bytes.
- Segmentation budgets based on GSM mobile user-data capacity, distinct from SMPP `short_message` field limit: single GSM7 160 septets / UCS2 70 BMP code units / binary 140 octets. Default Auto UDH reference is **16-bit**: `06 08 04 ref_hi ref_lo total part`, `esm_class` UDHI set; multipart GSM7 152 septets, UCS2 66 code units, binary 133 octets per part. Optional 8-bit UDH compatibility mode `05 00 03 ref total part` uses 153/67/134 respectively. Never split inside a GSM escape pair or UCS2 code unit; maximum 255 parts. References are constant across a logical SMS and must minimize reuse for the same destination; expose collision risk at extreme traffic rates, do not claim impossibility.
- Manual SAR TLVs use `sar_msg_ref_num` (16-bit), `sar_total_segments` and `sar_segment_seqnum` (1..255) on each segment, with no UDH/UDHI. Do not combine SAR and UDH. Use SMS user-data budgets, not the 254-octet SMPP field maximum, for network-targeted segments.
- "No automatic splitting" sends one unmodified `short_message` only when encoded length <=254 octets; warn when it exceeds the usual single-SMS network capacity. Greater than 254 octets => pre-send validation error unless user explicitly selects Advanced `message_payload`; that TLV uses `sm_length=0` and empty `short_message`, is never silently selected and may be SMSC-dependent. Automatic modes must not silently use `message_payload`.

## Confirmed DLR correlation and lifecycle (D-033)

- Receive DLR via `deliver_sm` on RX/TRX independent of submit session. Classify from `esm_class` receipt indicator and/or coherent standard DLR TLVs; For **message ID**, first parse `id:` inside the DLR text in `short_message` (Appendix B); only if absent or unusable, fallback to the `receipted_message_id` TLV. If both IDs exist and disagree, **text ID wins for correlation** and both raw values plus a conflict diagnostic are recorded. For **delivery status**, prefer `message_state` TLV; fallback to text `stat:` only if TLV state is absent or unusable. Never silently normalize conflicting values. Non-DLR `deliver_sm` is ACKed and counted as ignored/unclassified input, **not** treated as an MO testing scenario.
- Default ID correlation is **exact opaque, case-sensitive matching** of the decoded C-octet message ID, across all RX/TX sessions. Do not guess decimal-to-hex conversion, trim significant zeros or change case. A per-target explicit compatibility option may specify a known canonicalization rule. Duplicate IDs producing more than one candidate are marked `ambiguous`, never arbitrarily assigned.
- One accepted `submit_sm_resp` with usable `message_id` creates one segment-level DLR correlator by default. Multipart logical outcome is derived from segment evidence; don't assume one receipt per logical SMS, and don't call an SMS delivered if some segments are rejected/unresolved. A target-specific, explicitly selected logical-level DLR expectation may be supported only with known vendor mapping; no guessing. Track `accepted_without_id` separately.
- Receipt eligibility: `registered_delivery` low bits `0x00` => none guaranteed; `0x01` => one final receipt **expected per accepted, identifiable submitted PDU**; `0x02` => receipt conditional on final failure (absence does not prove missing); `0x03` => reserved SMPP 3.4 vendor test (no guaranteed receipt). For raw overrides, account for nonstandard bits and compatibility warnings instead of assuming universal final DLR. Submit + DLR drain can finish early when all guaranteed receipts resolve, but if conditional (`0x02`) or vendor-specific (`0x03`) receipts were requested it observes until its bounded drain deadline to give those receipts a chance to arrive; such missing observations are **not** counted as missing DLRs. If no receipts of any kind were requested (`0x00`), do not wait unnecessarily.
- Terminal states: delivered only for `DELIVERED`/`DELIVRD`; failed for `EXPIRED`, `DELETED`, `UNDELIVERABLE`, `REJECTED` and matching text; `ENROUTE` intermediate; `ACCEPTED` and `UNKNOWN` tracked explicitly, not auto-counted as delivered. Preserve raw SMSC state and diagnostic error. Report `unmatched`, `duplicate`, `ambiguous`, `late`, `missing`, `unsolicited`; negative or uncertain submit outcomes are not presumed accepted.
- Default DLR-drain deadline **60s**, configurable and independent of 5s submit-response timeout. The timeout is an *observation window*, not proof of delivery failure; preserve late/unresolved classification and never retry ambiguous submits.

## Approved encoding policy (design, not implemented)

- **Default `Auto`:** classify the **fully expanded text** against the GSM 03.38 basic alphabet **and extension table**. If every character is representable, select GSM7; otherwise select UCS2 for representable characters (notably Persian). No silent lossy replacement or transliteration.
- **Manual `GSM7`, `UCS2`, `Binary`:** Advanced controls select the encoding explicitly. Binary consumes raw octets rather than applying text auto-detection. Expose `data_coding` explicitly for specialist protocol tests; preserve consistent encoded bytes and flag intentional encoding/DCS mismatch if permitted later.
- SMPP `data_coding=0x00` means **SMSC Default Alphabet** (commonly interpreted as GSM7 by interoperable implementations), not an unconditional on-wire claim that all SMSCs accept one GSM septet packing convention. `0x08` denotes UCS2. Per D-032 default SMPP GSM7 is represented as unpacked GSM alphabet octets; packed mode, if supported, needs explicit Advanced selection.
- Per D-032 non-BMP/surrogate input (including emoji) is rejected by default; no silent truncation or replacement.
- Length/segmentation uses encoded octets, GSM escape characters' actual lengths, UDH/SAR overhead, and the selected carrier format. Record encoding used, DCS sent, and PDU/segment counts in metrics/results.
- Apply the same rules through the Go API, CLI, Web UI, JSON/YAML headless runner and data templates. This section is a design choice, **not a finalized config schema**.

References for the technical distinction: SMPP v3.4 Issue 1.2 data_coding description and https://docs.tibco.com/pub/bwsmpp/6.0.3/doc/html/users-guide/message-encoding.htm; GSM 03.38 alphabet and UCS2 mapping: https://developers.sinch.com/docs/sms/smpp/message-encoding.

## Approved DLR request policy (design, not implemented)

- Default `submit_sm.registered_delivery=0x00`; Basic has a per-Run Request DLR opt-in. An explicit Advanced full-byte override takes precedence, supporting protocol-specific receipt flags.
- **Basic ON = `0x01`** (final success or failure), OFF = `0x00`; no other bits set in Basic.
- **Advanced `0x02`** requests a receipt on final failure only.
- **Advanced `0x03`** is explicitly selectable for testing vendor-specific success-only behavior. It is **reserved in SMPP v3.4**, not a standards-defined success-only option. Preserve the exact byte; show a warning; observe and report actual SMSC behavior without fallback or assumed receipts. The SMPP v3.4 bind is unchanged.
- Source: SMPP Developers Forum SMPP v3.4 Issue 1.2 §5.2.17, https://www.camera-sdk.com/attachments/3324/SMPP_v3_4_Issue1_2.pdf
- Requesting DLR is separate from receiving/ACKing `deliver_sm`, fault injection policy, and completion choice. Submit+DLR completion must not wait for receipts that were never requested; classify any unsolicited receipts separately.
- D-033 defaults to per-segment DLR correlation; logical-level SMSC mapping requires explicit vendor selection. Submit success never proves delivery.

## Approved TON/NPI selection (design, not implemented)

- **Basic default = Auto**; source address and destination address are classified independently.
- **Confirmed mapping (D-029):** clearly alphanumeric `MyBrand` -> `TON=5, NPI=0`; explicitly international `+989121234567` -> `TON=1, NPI=1`; all digit-only strings without leading `+` (e.g. `989121234567`, `1000`) -> `TON=0, NPI=0`.
- Digit-only inputs **without** `+` always use the conservative Unknown/Unknown `0/0` default, regardless of numeric length or apparent country prefix.
- **Advanced/Go/headless:** permit independent explicit overrides for `source_addr_ton`, `source_addr_npi`, `dest_addr_ton`, `dest_addr_npi`; manual fields take precedence over Auto selection.
- Crucial invariant: TON/NPI is a **PDU metadata decision**. The original configured `source_addr` and `destination_addr` values must be transmitted **without implicit modification**. In particular, the fate of an optional leading `+` belongs to a separate explicit user-controlled normalization policy, if ever approved, not to Auto TON/NPI.
- Expose resolved TON/NPI and the unchanged addresses in full-detail and structured diagnostics for repeatable C++/Go integration tests. Avoid mandatory full capture in high-performance mode.
- Ordinary leading-plus, numeric-only, and short-code mappings are approved. D-032 fixes strict Basic ASCII address validation, with explicit Advanced exceptions; no implicit address rewriting is allowed.

## Approved Multipart SMS policy (design, not implemented)

- **Default = Auto:** messages requiring segmentation are automatically split with **UDH**, preserving original logical-message identity.
- **Advanced overrides:** user may explicitly choose `UDH`, `SAR TLVs`, or `No automatic splitting`. For an already-short message, do not add multipart overhead unnecessarily.
- For UDH segments, document the appropriate UDHI indication in `esm_class`, consistent concatenation reference, total parts/part index, and correctly encoded segment payload. For SAR segments, specify the relevant `sar_msg_ref_num`, `sar_total_segments`, `sar_segment_seqnum` TLVs and validate their consistent values.
- Count and report **logical messages separately from physical `submit_sm` PDUs/segments**. Each segment has an independent `submit_sm_resp`; never claim all multipart segments succeeded when only some were accepted. D-033 resolves multipart DLR correlation by default per accepted identifiable segment; logical-level vendor conventions require explicitly configured vendor mapping.
- All limits must derive from **actual encoded bytes/septets and multipart overhead**, not character count. D-032 fixes size budgets, default 16-bit and optional 8-bit UDH, explicit `message_payload` and unpacked GSM7 wire baseline; SMSC interoperability still requires tests.
- **No automatic splitting** is intended for controlled protocol testing, **not permission for truncation**. D-032 requires pre-send rejection unless Advanced `message_payload` is explicitly requested. Do not silently replace, truncate or auto-convert a payload.
- Same configuration and behavior are used by CLI, embedded Web UI, Go API and JSON/YAML headless runner. Config key names remain undecided.

DLR processing: distinguish `submit_sm_resp` acceptance from terminal delivery; extract the DLR `message_id` text-first (`id:`), then TLV fallback, correlate with returned submit message ID (and context if necessary), and track delivered/failed/unknown, missing, late, duplicate and unmatched receipts. DLR may arrive over a different RX/TRX session from original submit. D-033 fixes DLR message ID text `id:` first, TLV fallback and opaque exact matching; vendor ID normalization is explicit, never guessed.

## Data generator

Input alternatives: fixed/list/CSV, incremental number range, seeded random prefix/range, weighted source or destination distribution. Text content can be fixed, generated/random, or template with fields such as `{{sequence}}`, `{{destination}}`, and `{{run_id}}`. Generate lazily, not in a million-message prebuilt slice.

Finite CSV/list data uses two independent per-run switches:
- Exhaustion: `stop` or `loop`.
- Traversal: `sequential` or `shuffle`.

Allow deterministic seed, bounded memory, and disk-backed index or equivalent for large-file shuffle. Stop at dataset exhaustion must be reported distinctly.

## Response fault injection — confirmed

Goal: deliberately control the SMPP **client reply** to received `deliver_sm` (primarily DLR) to test how the **SMSC** responds to ACK errors, delays, or missing replies.

- A generic `deliver_sm` response policy may define weighted `command_status` failures, delayed responses, and intentionally **no response**.
- A DLR-specific policy **overrides** the generic policy when a receipt is classified as DLR.
- Each configured failure has a percentage; the remainder is success `ESME_ROK`. Sum of mutually exclusive action percentages may not exceed 100%.
- Example statuses: `ESME_RX_T_APPN`, `ESME_RX_P_APPN`, `ESME_RX_R_APPN`, `ESME_RSYSERR`. Validate status values against supported SMPP definitions before implementation.
- Randomized outcomes can use a configurable deterministic seed for reproducible tests; percentage means selection probability, not guaranteed exact quota for small samples.
- Preserve response PDU sequence number and proper framing. Do not confuse negative `deliver_sm_resp` with actual SMSC delivery failure.
- Track per-policy selected, emitted success/error, delayed, deliberately unanswered, socket send failure, and DLR classification counters; sampling must not distort totals.
- Avoid unbounded delay queues or goroutines. Do not implement packet corruption/network injection unless separately approved.

Illustrative only — **NOT A FINAL YAML SCHEMA**:

```yaml
inbound_faults:
  seed: 12345
  deliver_sm:
    errors:
      - {status: ESME_RX_T_APPN, percent: 5}
      - {status: ESME_RSYSERR, percent: 3}
    no_response_percent: 2
  dlr_override:
    errors:
      - {status: ESME_RX_T_APPN, percent: 10}
      - {status: ESME_RX_P_APPN, percent: 5}
    no_response_percent: 2
```

For the example DLR override, 83% success, 10% temporary app error, 5% permanent app error and 2% no response are target selection probabilities. Exact delay syntax, allowed codes, precedence of delayed errors, and behavior for inbound non-DLR `deliver_sm` outside MO scope remain **open for design**.

## Capture and completion

Full-detail capture and high-performance/sampled capture share accurate aggregate statistics; DLR correlation remains available even without full historical raw PDU capture. Captured details have explicit disk and retention limits.

Completion is configurable:
- `submit`: finish after submit generation and bounded submit-response drain; report DLR observed so far, without treating not-yet-due DLRs as failures.
- `submit_and_dlr`: after submit drain, continue DLR intake until configured deadline or expected receipts resolve; report missing/unmatched/duplicate/late statuses.

For a `submit_sm_resp` not known accepted, do not automatically count an expected DLR without reliable evidence; D-033 defines expected receipt eligibility; do not assume receipts for uncertain or negative submit outcomes.
