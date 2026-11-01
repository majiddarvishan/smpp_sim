# SMPP Protocol, Datasets, DLR and Fault Injection

## Protocol scope

V1 supports plain TCP SMPP client binds TX/RX/TRX, sends `submit_sm` and consumes `submit_sm_resp`. It receives delivery receipts as inbound `deliver_sm` and sends matching `deliver_sm_resp` (same request `sequence_number`). **Delivery report is not an independent SMPP command**. Do not implement MO test flows.

Required submit/message fields include source/destination TON/NPI, addresses, `data_coding`, `esm_class`, `registered_delivery`, message body or payload and optional TLVs. Include GSM7/UCS2/binary and multipart UDH/SAR paths as supported by the target SMPP profile. Default encoding **selection** and multipart mode selection are confirmed as Auto defaults (see below); exact encoding boundaries, validity formatting, TLV catalog, multipart framing/reference and no-split overlength behavior still require protocol-focused design review.

## Approved encoding policy (design, not implemented)

- **Default `Auto`:** classify the **fully expanded text** against the GSM 03.38 basic alphabet **and extension table**. If every character is representable, select GSM7; otherwise select UCS2 for representable characters (notably Persian). No silent lossy replacement or transliteration.
- **Manual `GSM7`, `UCS2`, `Binary`:** Advanced controls select the encoding explicitly. Binary consumes raw octets rather than applying text auto-detection. Expose `data_coding` explicitly for specialist protocol tests; preserve consistent encoded bytes and flag intentional encoding/DCS mismatch if permitted later.
- SMPP `data_coding=0x00` means **SMSC Default Alphabet** (commonly interpreted as GSM7 by interoperable implementations), not an unconditional on-wire claim that all SMSCs accept one GSM septet packing convention. `0x08` denotes UCS2. Decide packing/byte representation per SMPP binding and interoperability tests before code.
- Exact handling of text characters outside the UCS-2 Basic Multilingual Plane (e.g., emoji) is **unresolved**. Such text must **not** be silently truncated, replaced, or mislabelled as UCS2.
- Length/segmentation uses encoded octets, GSM escape characters' actual lengths, UDH/SAR overhead, and the selected carrier format. Record encoding used, DCS sent, and PDU/segment counts in metrics/results.
- Apply the same rules through the Go API, CLI, Web UI, JSON/YAML headless runner and data templates. This section is a design choice, **not a finalized config schema**.

References for the technical distinction: SMPP v3.4 Issue 1.2 data_coding description and https://docs.tibco.com/pub/bwsmpp/6.0.3/doc/html/users-guide/message-encoding.htm; GSM 03.38 alphabet and UCS2 mapping: https://developers.sinch.com/docs/sms/smpp/message-encoding.

## Approved TON/NPI selection (design, not implemented)

- **Basic default = Auto**; source address and destination address are classified independently.
- Clear **alphanumeric** address presentation can select `TON=5, NPI=0`. Explicit international-address indication can select `TON=1, NPI=1`. These are examples of intended classification, **not a finalized parser** or a promise that numeric-only input is always international.
- In ambiguous cases, choose a conservative default; exact values and parsing precedence are **not yet approved**. A number prefix or digit string alone must not trigger unapproved rewriting or normalization.
- **Advanced/Go/headless:** permit independent explicit overrides for `source_addr_ton`, `source_addr_npi`, `dest_addr_ton`, `dest_addr_npi`; manual fields take precedence over Auto selection.
- Crucial invariant: TON/NPI is a **PDU metadata decision**. The original configured `source_addr` and `destination_addr` values must be transmitted **without implicit modification**. In particular, the fate of an optional leading `+` belongs to a separate explicit user-controlled normalization policy, if ever approved, not to Auto TON/NPI.
- Expose resolved TON/NPI and the unchanged addresses in full-detail and structured diagnostics for repeatable C++/Go integration tests. Avoid mandatory full capture in high-performance mode.
- Finalize parser treatment of a leading `+`, all-digit international-form numbers with no `+`, short codes, prefixes and non-alphanumeric special characters before implementation.

## Approved Multipart SMS policy (design, not implemented)

- **Default = Auto:** messages requiring segmentation are automatically split with **UDH**, preserving original logical-message identity.
- **Advanced overrides:** user may explicitly choose `UDH`, `SAR TLVs`, or `No automatic splitting`. For an already-short message, do not add multipart overhead unnecessarily.
- For UDH segments, document the appropriate UDHI indication in `esm_class`, consistent concatenation reference, total parts/part index, and correctly encoded segment payload. For SAR segments, specify the relevant `sar_msg_ref_num`, `sar_total_segments`, `sar_segment_seqnum` TLVs and validate their consistent values.
- Count and report **logical messages separately from physical `submit_sm` PDUs/segments**. Each segment has an independent `submit_sm_resp`; never claim all multipart segments succeeded when only some were accepted. DLR correlation for multipart is a separate unresolved detail, especially when SMSCs report per segment versus per logical message.
- All limits must derive from **actual encoded bytes/septets and multipart overhead**, not character count. Exact thresholds, UDH reference width/rollover and compatibility, `message_payload` use, and GSM7 wire representation remain for protocol-design review.
- **No automatic splitting** is intended for controlled protocol testing, **not permission for truncation**. Whether oversized payloads are rejected pre-send or represented using a separately explicit `message_payload` option remains to be approved. Do not silently replace, truncate or auto-convert a payload.
- Same configuration and behavior are used by CLI, embedded Web UI, Go API and JSON/YAML headless runner. Config key names remain undecided.

DLR processing: distinguish `submit_sm_resp` acceptance from terminal delivery; correlate using the returned message ID (and context when necessary); track delivered/failed/unknown, missing, late, duplicate and unmatched receipts. DLR may arrive over a different RX/TRX session from original submit. Message ID normalization rules are a **pending decision** (e.g. format differences across SMSCs).

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

For a `submit_sm_resp` not known accepted, do not automatically count an expected DLR without reliable evidence; specification of eligible expected receipts is a pending implementation detail.
