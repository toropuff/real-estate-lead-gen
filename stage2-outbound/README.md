# Stage 2 — Outbound AI Calling (Sergio Persona)

This folder contains everything that was added for the **Stage 2 outbound calling demo** for Sergio Tapia at West Capital Lending. Stage 1 was the inbound "Alex" agent (see root `CLAUDE.md` / `SETUP.md`). Stage 2 adds:

1. A Google Sheets–based lead pipeline that can be swapped for Airtable / CSV / any API later without touching the rest of the system.
2. A new n8n workflow (`Vapi — WCL Outbound Dialer`) that reads the pipeline, filters ready-to-dial rows, validates phones, and fires outbound VAPI calls.
3. A **new VAPI assistant** — *Sergio Tapia (Outbound Broker)* — distinct from Alex. Sergio speaks in first person, never mentions "West Capital Lending" by name, and runs a 7-phase consultative script ending in four cross-sell discovery questions.
4. A new VAPI structured output (*Sergio Outbound Extras*) capturing the four cross-sell answers + a concrete `next_step_action` enum.
5. An updated inbound parser workflow that now also writes every call result (inbound AND outbound) to the Google Sheet, keyed by `vapi_call_id`.

---

## Architecture

```
                         ┌─────────────────────────┐
                         │  Google Sheet (lead     │
                         │  pipeline, 58 cols)     │
                         └──────────┬──────────────┘
                                    │ read
                                    ▼
    Manual Trigger ─► Read Sheet ─► Filter Ready to Dial
                                        (NANP phone validation,
                                         retry window check)
                                    │
                                    ▼
                         Split In Batches (size=1)
                                    │
                                    ▼
                         Build VAPI Payload
                            (dynamic firstMessage,
                             full variableValues for
                             Sergio assistant)
                                    │
                                    ▼
                         VAPI Start Call (POST /call/phone)
                                    │
                                    ▼
                         Mark Row In Progress (update sheet)
                                    │
                                    ▼
                         Brief Pause (3s) → loop

                    Call happens on Sergio's side …

                         ┌────────────────────────────┐
                         │  end-of-call-report webhook │
                         └──────────┬──────────────────┘
                                    ▼
                    Vapi — WCL Lead Intake Handler
                    (Parse & Build Email)
                        ├─► Resend (email Chris)
                        └─► Google Sheets (append/update
                            by vapi_call_id, all 58 cols
                            including 4 cross-sell + next_step)
```

Inbound ("Alex") vs outbound ("Sergio") is disambiguated by `call.assistantOverrides.metadata.source` (`outbound_dialer` vs missing).

---

## Components

### VAPI

| Thing | ID |
|---|---|
| Inbound assistant "Alex" | `12b07537-547c-4fb5-bb5a-8a7ab0c1759d` |
| **Outbound assistant "Sergio Tapia (Outbound Broker)"** | `df1ccf6d-61e6-49de-a179-3d8ba63815b2` |
| Phone number (shared, inbound + caller ID for outbound) | `905a5925-e228-4144-bcac-3e4fa81f58c4` (`+1 949-942-8395`) |
| Structured output — WCL Loan Intake Details | `a10ce184-c397-49f0-94b2-da862db33833` |
| Structured output — Call Summary | `909a3bf7-2095-4079-959c-86f044477b57` |
| Structured output — Customer Sentiment | `e772b28f-3cfb-497a-83bd-4480e8fa3bb6` |
| Structured output — Success Eval (Pass/Fail) | `f80e08e9-ae03-4d13-a05c-65990a3e36c4` |
| Structured output — Success Eval (1–10) | `89ec4bda-bb11-4132-ad5b-99552774f175` |
| **Structured output — Sergio Outbound Extras** | `24d49be3-9b17-4941-871b-c8fd26a6f02e` |

The Sergio assistant is attached to **all six** structured outputs. Alex keeps the first five.

### n8n

| Workflow | ID | Notes |
|---|---|---|
| Vapi — WCL Outbound Dialer | `e5OaddLYhhV0aS2Z` | Manual trigger, reads sheet, dials one at a time |
| Vapi — WCL Lead Intake Handler | `MDXGW2s91yXtjyT7` | Webhook receiver, emails Chris, writes sheet row |

### Google Sheet

- **URL:** https://docs.google.com/spreadsheets/d/1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc/edit
- **Sheet ID:** `1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc`
- **gid:** `161758208`
- Columns cover: lead identity, call metadata, loan classification, DSCR/Fix-Flip detail, retry tracking, **Sergio-specific context (state, loan_amount_requested, inquiry_source, inquiry_date)**, call outcome, **cross-sell capture (4 fields)**, **next-step action + notes**, CRM sync.

---

## Sergio Persona — Key Rules

Full system prompt is in `artifacts/vapi-sergio-assistant.json` (`model.messages[0].content`). Highlights:

- **Name:** Sergio Tapia. Pronounced **SUR-jee-oh** (soft G).
- **Never** say "West Capital Lending" or any lender brand by name. He's a broker who shops deals.
- **Do not re-ask** for any field already populated in `variableValues` (name, phone, email, state, property type, loan type, transaction type, loan amount, FICO, inquiry source/date).
- Opening references the known inquiry: *"Hi [name], Sergio Tapia here. I'm looking at your inquiry from [source] regarding [amount] for your [property] in [state]. I see you've got a [FICO] and you're looking to [transaction]."*
- 7 phases: Personalized Opening → Position ("I specialize in F&F, GUC, DSCR") → Build Confidence → Targeted Intake (branches by loan type) → Soft Close + text the Loanzify portal link → **Cross-Sell Discovery (4 questions)** → Warm Close.
- Offers to **text** the Loanzify URL (`westcaplending.loanzify.io/register/sergio-tapia`); never reads it aloud.
- TTS pronunciation rules: spell out dollar amounts, percentages, and acronyms (D-S-C-R, L-T-V, A-R-V, F-I-C-O). No markdown.
- If asked for rates or guarantees → deflects ("I'll shop it and come back with real numbers").

### The four cross-sell questions (captured in Sergio Outbound Extras)

1. **xsell_walked_away_deal** — recently walked from a deal due to low appraisal / insufficient leverage?
2. **xsell_stuck_project** — project stuck mid-reno because a lender stopped funding draws or changed terms?
3. **xsell_bridge_to_dscr_refi** — stabilized rentals still on high-interest bridge debt that should refi to 30-yr DSCR?
4. **xsell_capital_locked_estimate** — roughly how much capital is locked up in the current portfolio?

Plus **next_step_action** (enum): `Text Loanzify Link`, `Email Loanzify Link`, `Send Pre-Flight Analysis`, `Callback Scheduled`, `Awaiting Prospect Info`, `No Interest`, `Unable To Qualify`, `Other`, `Unknown`.

---

## How It Gets Dialed

The **Build VAPI Payload** node constructs a dynamic `firstMessage` based on whatever fields are populated on the row, and passes ALL known context via `assistantOverrides.variableValues`:

```
first_name, last_name, phone, email, state, property_type,
known_loan_type, transaction_type, loan_amount_requested,
fico, inquiry_source, inquiry_date
```

Helpers in the node:
- `numToWords(n)` — 0..999 → English words (for dollar spell-out)
- `spellMoney(raw)` — `310000` → `"three hundred ten thousand dollars"` (so the firstMessage sounds natural through TTS without relying on formatPlan at the template boundary)

The **Filter Ready to Dial** node does strict NANP validation before dialing so we never burn a call on a truncated / typo'd number:
- Must start with `+`
- If `+1` (NANP): exactly 11 digits, area code and exchange first-digit 2–9, no N11 reserved codes
- Other countries: general E.164 (8–15 digits)
- Every skipped row logs a reason (`missing_plus_prefix`, `nanp_must_be_11_digits_got_10`, etc.)
- **All row values are coerced to String first** (Google Sheets returns numeric cells as JS numbers, which have no `.trim()`)

---

## Running a Test Call

1. Open the sheet, add a row with your own number and at minimum: `first_name`, `phone` (E.164, e.g. `+14155551234`), `call_status = Pending`, plus as many context fields as you want populated (`state`, `loan_type`, `transaction_type`, `loan_amount_requested`, `ff_credit_score`, `inquiry_source`, `inquiry_date`).
2. Ensure the `phone` cell is **Plain text format** in Google Sheets (Format → Number → Plain text), otherwise the leading `+` is stripped.
3. n8n → **Vapi — WCL Outbound Dialer** → Execute Workflow.
4. Sergio should open with the personalized reference. Run the script, say yes to the text link, answer some cross-sell questions, hang up.
5. Within ~15s the row repopulates with transcript, summary, cross-sell answers, and `next_step_action`.

---

## Known Deferred Items

- **Caller ID shows as "+94" (Sri Lanka) on some carriers.** VAPI's shared SIP trunk drops the leading `+1` when presenting outbound caller ID. Fix = bring-your-own-Twilio number (deferred to production rollout).
- **Professional Voice Clone of Sergio** — currently using ElevenLabs Liam (`TX3LPaxmHKxFdv7VOQHJ`) as a placeholder for the demo. Production will swap in Sergio's own clone.
- **Twilio Lookup for reachability validation** — currently only format-validating. Could add a pre-dial carrier/line-type lookup to skip disconnected/non-mobile numbers.

---

## Artifacts

All live n8n workflows and VAPI assistants/structured outputs are snapshotted under [`/infra`](../infra/) at the repo root. Refresh them any time with `./infra/sync-infra.sh`. Relevant files for Stage 2:

| File | What it is |
|---|---|
| [`infra/n8n/vapi-wcl-outbound-dialer.json`](../infra/n8n/vapi-wcl-outbound-dialer.json) | Full export of the outbound dialer workflow (`e5OaddLYhhV0aS2Z`) |
| [`infra/n8n/vapi-wcl-lead-intake-handler.json`](../infra/n8n/vapi-wcl-lead-intake-handler.json) | Updated inbound handler (`MDXGW2s91yXtjyT7`) with sheet-sync + Sergio extras parsing |
| [`infra/vapi/assistants/sergio-outbound-broker.json`](../infra/vapi/assistants/sergio-outbound-broker.json) | Full VAPI config for the Sergio Tapia outbound assistant |
| [`infra/vapi/assistants/alex-inbound.json`](../infra/vapi/assistants/alex-inbound.json) | Full VAPI config for the Alex inbound assistant (for comparison) |
| [`infra/vapi/structured-outputs/sergio-outbound-extras.json`](../infra/vapi/structured-outputs/sergio-outbound-extras.json) | Schema for the Sergio Outbound Extras structured output |
