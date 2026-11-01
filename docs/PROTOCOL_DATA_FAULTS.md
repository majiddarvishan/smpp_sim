# SMPP Protocol, Datasets, DLR and Fault Injection

## Protocol scope

V1 supports plain TCP SMPP client binds TX/RX/TRX, sends `submit_sm` and consumes `submit_sm_resp`. It receives delivery receipts as inbound `deliver_sm` and sends matching `deliver_sm_resp` (same request `sequence_number`). **Delivery report is not an independent SMPP command**. Do not implement MO test flows.

Required submit/message fields include source/destination TON/NPI, addresses, `data_coding`, `esm_class`, `registered_delivery`, message body or payload and optional TLVs. Include GSM7/UCS2/binary and multipart UDH/SAR paths as supported by the target SMPP profile. Encoding boundaries, defaults, validity formatting, TLV catalog and multipart method selection require protocol-focused design review.

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
