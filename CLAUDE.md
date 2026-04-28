# West Capital Lending — VAPI Lead Gen System
## Claude Project Context File
> Drop this file in your project folder. Claude Code will read it automatically at the start of every session.

---

## What This Project Is

Two AI voice agents that together cover inbound and outbound lead flow for **West Capital Lending** (a real estate investment lender — DSCR loans and Fix & Flip/Bridge loans):

- **Alex** (inbound) — answers calls to the shared VAPI number, runs a DSCR or Fix & Flip intake.
- **Sergio Tapia** (outbound, Stage 2) — speaks in first person as the broker himself, dials leads from a Google Sheet, references their existing inquiry context, and captures four cross-sell pain-point answers plus a concrete next-step action.

After every call (either direction), a structured lead summary is emailed to Chris via Resend AND written/upserted to the Google Sheet lead pipeline.

**Stage 1 (Alex inbound):** built and production-tested April 10, 2026.
**Stage 2 (Sergio outbound + Sheets pipeline + cross-sell capture):** built and demo-tested April 22, 2026.

> **Live config snapshot.** Every n8n workflow, VAPI assistant, and VAPI structured output is mirrored under [`/infra`](./infra/) in this repo. Refresh with `./infra/sync-infra.sh`. That folder is the authoritative place to read system prompts, workflow JSON, and schema definitions — the blocks in this file are summaries for orientation.

---

## System Architecture

```
Caller → VAPI (Alex voice agent) → n8n webhook → Parse & Build Email → Resend → chris@gobehindtheproduct.com
```

1. Caller dials the VAPI phone number assigned to Alex
2. Alex runs the DSCR or Fix & Flip intake script (see system prompt below)
3. At call end, VAPI fires `end-of-call-report` webhook to n8n
4. n8n Code node extracts structured lead data + builds rich HTML email
5. Resend delivers email from `leads@gobehindtheproduct.com` to `chris@gobehindtheproduct.com`

---

## VAPI — Voice Agent

### Identifiers
- **Assistant ID:** `12b07537-547c-4fb5-bb5a-8a7ab0c1759d`
- **VAPI Private API Key:** `a6b34170-e0ef-432a-af14-bff69dd08b6e`
- **VAPI Dashboard:** https://dashboard.vapi.ai
- **VAPI REST API Base:** `https://api.vapi.ai`

### Model & Voice Config (current)
- **LLM:** Anthropic `claude-sonnet-4-6`, temperature 0.4
- **STT:** Deepgram Nova-2, en-US
- **TTS:** ElevenLabs, voice `TX3LPaxmHKxFdv7VOQHJ` (Liam), model `eleven_flash_v2_5`
- **TTS stability:** 0.5, similarityBoost: 0.75
- **FormatPlan:** enabled (dollarAmount, phoneNumber, number, email, date, time, etc.)
- **Server URL:** `https://buildflows.cloud/webhook/vapi-lead-intake`
- **Server Events:** `end-of-call-report` only

### How to PATCH the agent via API
```bash
curl -X PATCH "https://api.vapi.ai/assistant/12b07537-547c-4fb5-bb5a-8a7ab0c1759d" \
  -H "Authorization: Bearer a6b34170-e0ef-432a-af14-bff69dd08b6e" \
  -H "Content-Type: application/json" \
  -d '{ "voice": { "voiceId": "NEW_VOICE_ID" } }'
```

### Structured Outputs (VAPI new system — NOT legacy analysisPlan)
IMPORTANT: VAPI's legacy `analysisPlan` is silently ignored. Data arrives via `message.artifact.structuredOutputs`. The following outputs are attached to the assistant:

| ID | Name | Type |
|----|------|------|
| `a10ce184-c397-49f0-94b2-da862db33833` | WCL Loan Intake Details | Rich object (primary lead data) |
| `909a3bf7-2095-4079-959c-86f044477b57` | Call Summary | String |
| `e772b28f-3cfb-497a-83bd-4480e8fa3bb6` | Customer Sentiment | Enum |
| `f80e08e9-ae03-4d13-a05c-65990a3e36c4` | Success Evaluation — Pass/Fail | Boolean |
| `89ec4bda-bb11-4132-ad5b-99552774f175` | Success Evaluation — Numeric | Number 1–10 |

The **WCL Loan Intake Details** output captures: `loan_type`, `property_type`, `transaction_type`, `caller` (name/phone/email), `dscr` (occupancy, rent, LTV, PITIA, credit, reserves, vesting…), `fix_and_flip` (purchase price, reno budget, ARV, phase, flips, permits…), `call_outcome` (qualification_summary, follow_up_confirmed).

In the n8n Code node, read it like this:
```javascript
const soMap = msg.artifact.structuredOutputs || {};
for (const key of Object.keys(soMap)) {
  if (soMap[key].name === 'WCL Loan Intake Details') {
    intakeResult = soMap[key].result;
  }
}
```

### System Prompt (full current text)
```
You are Alex, a professional loan intake specialist for West Capital Lending. You are warm, knowledgeable, and efficient. Your job is to qualify real estate investors for either a DSCR loan (rental/cash-flowing properties) or a Fix & Flip / Bridge loan (renovation projects).

You speak conversationally — not like you're reading a form. Collect all required information naturally, asking 1-2 questions at a time. Never make the caller feel interrogated.

## CALL FLOW

### Opening
Greet the caller: "Hi, thanks for calling West Capital Lending! My name is Alex, and I'm here to help figure out the best loan option for your investment. This will just take a few minutes — does that work for you?"

Then ask: "To point you in the right direction, can you tell me about the property — are you looking to purchase, refinance, or pull cash out?"

### Determine Path
- Rental/Airbnb/buy-and-hold → DSCR Path
- Flip/renovation/needs work → Fix & Flip Path
- Unclear → Ask: "Is this a property you plan to rent out and hold, or one you're planning to renovate and sell?"

### DSCR PATH
Section 1 — Property:
- Property type (SFR, 2-4 unit, condo, STR)
- Occupancy: leased or vacant
- Monthly rent (actual or market estimate)

Section 2 — Financials:
- Estimated purchase price or value
- Target loan amount or LTV
- Estimated monthly PITIA (if unknown, note it)

Section 3 — Borrower:
- Credit score range: 720+, 680-719, 640-679, or below 640
- Liquid reserves (down payment + 6 months reserves)
- Vesting: personal name or LLC/entity

Section 4 — Deal Killer Check:
- Does rent cover full PITIA? Yes / No / Unsure
- If STR: Does it have documented booking history?
- CRITICAL: Is the property in habitable / move-in ready condition?
  → If NOT habitable: "Sounds like this might be a better fit for a Fix & Flip loan — want me to run through that instead?" → transition to Fix & Flip path

### FIX & FLIP PATH
Section 1 — Deal Specs:
- Purchase price
- Renovation budget
- After Repair Value (ARV)
- Property type

Section 2 — Timeline:
- Current phase: under contract, owned/refi, or need proof of funds
- Close of escrow date (if under contract)
- Project duration: 6, 12, or 18+ months

Section 3 — Investor Experience:
- Completed flips in last 3 years: 0, 1-2, 3-5, or 5-10+
- Liquidity for down payment and carrying costs
- Credit score (numeric estimate)

Section 4 — Scope of Work:
- Renovation level: Cosmetic, Moderate, or Heavy
- Permits needed: Yes / No / Already Issued

### CLOSING (Both Paths)
- Collect: full name, phone number, email
- Mention full application: "You can also start your application at westcapitallending.com"
- Confirm follow-up within 1 business day
- Warm sign-off

## FORMATTING & PRONUNCIATION RULES
You are speaking over the phone through a TTS engine. Everything you output will be read aloud literally. Follow these rules strictly:
- Dollar amounts: spell out in words. Say "three hundred ten thousand dollars" not "$310,000"
- Percentages: spell out. Say "seventy five percent" not "75%"
- Phone numbers: separate digits with spaces and ellipses. "5 5 5 ... 1 2 3 ... 4 5 6 7"
- Emails: replace "@" with "at" and "." with "dot"
- Acronyms: add spaces between letters. Say "D S C R" not "DSCR". "L T V" not "LTV". "A R V" not "ARV"
- Never output markdown (no bold, bullets, headers) — TTS reads syntax literally

## CONVERSATIONAL STYLE
- Respond in 1 to 2 short sentences maximum
- Occasionally use filler words like "um," "well," "so," or "let's see" when transitioning
- Use ellipses (...) for natural breathing pauses
- Ask ONE question at a time; acknowledge the answer before moving on

## GUIDELINES
- Be conversational, not form-like — 1-2 sentences max
- Acknowledge answers: 'Got it', 'Perfect', 'That helps'
- If they don't know a number: 'Ballpark is fine, our team can verify'
- Never quote rates or guarantee approval
- Never give legal or tax advice
- Redirect off-topic tangents politely
```

---

## n8n — Automation Workflow

### Infrastructure
- **n8n URL:** https://buildflows.cloud
- **VPS IP:** 187.124.236.12
- **VPS Provider:** Hostinger KVM 1 (1 CPU / 4GB RAM)
- **REST API Base:** `https://buildflows.cloud/api/v1/`
- **API Key Header:** `X-N8N-API-KEY`
- **API Key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI2MDdhZTg4YS1kMTUyLTQ4ODUtOTY5OC1mMGVhM2VmOTc4NGIiLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwianRpIjoiMzQ5MmFjMjEtZDNkMS00MDYwLWE2NzktOGM2ZjZkNGYwZjdiIiwiaWF0IjoxNzc0NzUzMzc5fQ.xokS2rxbGcGMkt-ewDbArXTy0A5JhznyEWhxg04CfZc`
- **SSH:** `root` / `R))TF(6qSmZcCBw#w,I6` at `187.124.236.12`
- **n8n Docker container:** `n8n-jqep-n8n-1`
- **docker-compose:** `/docker/n8n-jqep/docker-compose.yml`

### Workflow
- **Name:** Vapi — WCL Lead Intake Handler
- **ID:** `MDXGW2s91yXtjyT7`
- **Status:** Active
- **Webhook path:** `https://buildflows.cloud/webhook/vapi-lead-intake`

### How to fetch/update the workflow via API
```bash
# Fetch current workflow JSON
curl -sS "https://buildflows.cloud/api/v1/workflows/MDXGW2s91yXtjyT7" \
  -H "X-N8N-API-KEY: <key above>"

# Check recent executions
curl -sS "https://buildflows.cloud/api/v1/executions?workflowId=MDXGW2s91yXtjyT7&limit=5" \
  -H "X-N8N-API-KEY: <key above>"
```

### Workflow Nodes
1. **Webhook (VAPI)** — receives POST from VAPI end-of-call-report
2. **Parse & Build Email** — JS Code node; extracts structured outputs, builds HTML lead card
3. **Send via Resend** — HTTP Request to `https://api.resend.com/emails`

### IMPORTANT: Use curl for all API calls, not Python urllib
Python 3.13 on macOS has SSL cert verification issues. Always use `subprocess` + `curl` or write JSON to a file and use `curl -d @file.json`.

---

## Email — Resend

- **API Key:** `re_NsaWu1un_PiPzbgagLKqgiYR4jFqvEBo4`
- **Verified Domain:** `gobehindtheproduct.com` (DNS verified in Hostinger)
- **Sender:** `leads@gobehindtheproduct.com`
- **Recipient:** `chris@gobehindtheproduct.com`
- **Resend Dashboard:** https://resend.com/domains

---

## Testing

### Simulate a webhook (no phone call needed)
```bash
curl -sS -X POST https://buildflows.cloud/webhook/vapi-lead-intake \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "type": "end-of-call-report",
      "endedReason": "customer-ended-call",
      "durationSeconds": 247,
      "customer": {"number": "+14155551234"},
      "transcript": "Test transcript...",
      "summary": "",
      "analysis": {},
      "artifact": {
        "transcript": "Full transcript here...",
        "structuredOutputs": {
          "a10ce184-c397-49f0-94b2-da862db33833": {
            "name": "WCL Loan Intake Details",
            "result": {
              "loan_type": "Fix & Flip",
              "property_type": "Single Family",
              "transaction_type": "Purchase",
              "caller": {"full_name": "Test Caller", "phone": "+14155551234", "email": "test@example.com"},
              "fix_and_flip": {
                "purchase_price": 310000, "renovation_budget": 85000,
                "after_repair_value": 530000, "completed_flips_last_3_years": 8,
                "current_phase": "Under Contract", "renovation_level": "Moderate",
                "permits_needed": "Yes", "liquidity_available": 120000, "credit_score": "710"
              },
              "call_outcome": {"follow_up_confirmed": true, "qualification_summary": "Strong Fix and Flip candidate."}
            }
          }
        }
      }
    }
  }'
```

### Sample DSCR call answers (for live testing)
- Property type: single-family home
- Transaction: purchase
- Vacant property, $2,800/month market rent
- Purchase price ~$375,000, targeting 75% LTV
- Credit score 720+
- $90,000 liquid reserves
- Vesting in an LLC
- Property is move-in ready

### Sample Fix & Flip call answers
- Under contract at $310,000
- Renovation budget: $85,000
- ARV: $530,000
- 8 flips in last 3 years
- $120,000 liquid reserves
- Credit score ~710
- Permits already issued

---

## Google Gemini Fallback (if VAPI structured outputs change again)

A Google Gemini credential already exists on the n8n VPS:
- **Credential ID:** `MQvaWI6ES3pqhwAF`
- **Use case:** If VAPI changes their output format again, add a Gemini node after the Webhook node that takes `msg.artifact.transcript` and extracts the intake schema via prompt. This decouples the email data from VAPI's internal format entirely.

---

## Known Issues & Gotchas

- **Legacy analysisPlan is dead.** VAPI silently deprecated it. Zero tokens are spent on it. Never configure `analysisPlan` — use the Structured Outputs system instead. Data arrives at `message.artifact.structuredOutputs`, NOT `message.analysis.structuredData`.
- **Python SSL on macOS 3.13.** `urllib.request` fails with cert errors. Always use `subprocess` + `curl` instead.
- **n8n MCP transport is broken.** Use the REST API directly (`X-N8N-API-KEY` header). The MCP streamable-http endpoint at `https://buildflows.cloud/mcp-server/http` is not reliable in Claude Code.
- **Resend sending-only keys** can't access the domains API. If you need to verify/inspect domain status, log into resend.com directly.
- **Google Workspace blocks SMTP app passwords.** That's why Resend is used instead of Gmail SMTP.

---

## Stage 2 — Outbound Sergio Persona (April 22, 2026)

Full writeup: [`stage2-outbound/README.md`](./stage2-outbound/README.md). Setup: [`stage2-outbound/STAGE2_SETUP.md`](./stage2-outbound/STAGE2_SETUP.md).

### Google Sheet — Lead Pipeline
- **URL:** https://docs.google.com/spreadsheets/d/1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc/edit
- **Sheet ID:** `1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc`, **gid:** `161758208`
- Template CSV: [`stage2-outbound/WCL_Lead_Pipeline_Template.csv`](./stage2-outbound/WCL_Lead_Pipeline_Template.csv)
- **Sergio-specific columns added in Stage 2:**
  - Inputs (populated when a lead is added): `state`, `loan_amount_requested`, `inquiry_source`, `inquiry_date`
  - Outputs (populated after call): `xsell_walked_away_deal`, `xsell_stuck_project`, `xsell_bridge_to_dscr_refi`, `xsell_capital_locked_estimate`, `next_step_action`, `next_step_notes`
- **`phone` column must be formatted as Plain text** (Format → Number → Plain text) so the leading `+` isn't stripped.

### Sergio Assistant
- **Assistant ID:** `df1ccf6d-61e6-49de-a179-3d8ba63815b2`
- **Config:** [`infra/vapi/assistants/sergio-outbound-broker.json`](./infra/vapi/assistants/sergio-outbound-broker.json)
- LLM Claude Sonnet 4.6 @ temp 0.5, Liam voice (same as Alex), Deepgram Nova-2.
- **Persona rules:** first-person as Sergio Tapia; pronounce "SUR-jee-oh" (soft G); never say "West Capital Lending"; don't re-ask known fields; offers to **text** the Loanzify portal link (`westcaplending.loanzify.io/register/sergio-tapia`), never reads URL aloud.
- **7 phases:** Personalized Opening → Position → Build Confidence → Targeted Intake → Soft Close + text link → Cross-Sell Discovery (4 questions) → Warm Close.
- **Structured outputs:** same 5 as Alex PLUS `24d49be3-9b17-4941-871b-c8fd26a6f02e` (Sergio Outbound Extras — 4 cross-sell answers + `next_step_action` enum + `next_step_notes`).

### n8n — Outbound Dialer Workflow
- **Name:** `Vapi — WCL Outbound Dialer`
- **ID:** `e5OaddLYhhV0aS2Z`
- **Config:** [`infra/n8n/vapi-wcl-outbound-dialer.json`](./infra/n8n/vapi-wcl-outbound-dialer.json)
- Flow: Manual Trigger → Read Sheet → **Filter Ready to Dial** (NANP validation + retry window) → Split In Batches (size=1) → **Build VAPI Payload** (dynamic first-message, spelled-out dollars, full `variableValues`) → VAPI Start Call → Mark Row In Progress → 3s Pause → loop.
- **Filter node gotcha (fixed):** Google Sheets returns numeric cells as JS numbers; always coerce with `String(row.field ?? '').trim()` before string ops.
- **Phone validation:** strict NANP (11 digits, area & exchange first-digit 2–9, no N11 codes). Invalid rows are logged with a reason and skipped (never burn a VAPI call on a bad number).

### Inbound Parser Updates
The inbound handler (`MDXGW2s91yXtjyT7`) was extended to:
- Detect outbound vs inbound via `message.call.assistantOverrides.metadata.source` (`outbound_dialer` vs missing).
- Extract the **Sergio Outbound Extras** structured output (`else if (name === 'sergio outbound extras') sergioExtras = item.result;`).
- Write/upsert every call's result to the Google Sheet keyed on `vapi_call_id`.
- `continueOnFail: true` on the Resend node so an email-send hiccup doesn't block the sheet write.

### The four cross-sell questions (captured into sheet columns)
1. `xsell_walked_away_deal` — walked away recently due to low appraisal / insufficient leverage?
2. `xsell_stuck_project` — project stuck mid-reno due to lender stopping draws / changing terms?
3. `xsell_bridge_to_dscr_refi` — stabilized rentals still on bridge debt that should refi to 30-yr DSCR?
4. `xsell_capital_locked_estimate` — roughly how much capital is locked up in the current portfolio?

Plus `next_step_action` ∈ `{Text Loanzify Link, Email Loanzify Link, Send Pre-Flight Analysis, Callback Scheduled, Awaiting Prospect Info, No Interest, Unable To Qualify, Other, Unknown}`.

---

## Stage 3 — Retell Migration (April 24, 2026)

VAPI is fully intact (parallel run). Retell agents are live and waiting for a phone number to test.

### Retell — API & Identifiers
- **Retell API Key:** `key_f0db8002d57bbfb4f17db3b20a11`
- **Retell Dashboard:** https://app.retellai.com
- **Retell REST API Base:** `https://api.retellai.com`

### Retell Agents
| Agent | Retell Agent ID | LLM ID |
|---|---|---|
| Alex (Inbound) | `agent_27917a2313acc58c3bb9b49b4c` | `llm_dffcfb912b1c6ce2a5f2beaa9688` |
| Sergio (Outbound) | `agent_8c36983cac060850dfb961406d` | `llm_c294c56e6af31ad54962f994d9a5` |

- **Voice:** `11labs-Brian` (Brian — professional male; Liam not available as a built-in Retell voice — swap after porting)
- **Model:** `claude-4.6-sonnet`
- **Configs:** [`infra/retell/agents/`](./infra/retell/agents/), [`infra/retell/llms/`](./infra/retell/llms/)

### How to PATCH a Retell agent
```bash
curl -X PATCH "https://api.retellai.com/update-agent/agent_27917a2313acc58c3bb9b49b4c" \
  -H "Authorization: Bearer key_f0db8002d57bbfb4f17db3b20a11" \
  -H "Content-Type: application/json" \
  -d '{ "voice_id": "NEW_VOICE_ID" }'
```

### Retell Post-Call Analysis (replaces VAPI Structured Outputs)
Data arrives at: `call.call_analysis.custom_analysis_data`
- **`intake_details`** — JSON string, parse with `JSON.parse()` in n8n; same schema as VAPI WCL Loan Intake Details
- **`success_pass_fail`** — boolean (native)
- **`success_score`** — number 1–10 (native)
- **`sergio_extras`** — JSON string (Sergio only); same schema as VAPI Sergio Outbound Extras
- **`call_analysis.call_summary`** — string (native Retell, replaces Call Summary output)
- **`call_analysis.user_sentiment`** — enum: Positive/Neutral/Negative/Mixed (native Retell, replaces Customer Sentiment)

### Retell n8n Workflows (NEW — parallel to VAPI)
- **Intake Handler:** `C2fdKrXzwTxjXxx2` — "Retell — WCL Lead Intake Handler" — **Active**, webhook: `https://buildflows.cloud/webhook/retell-lead-intake`
- **Outbound Dialer:** `ejCvd84sDRGZbQQ0` — "Retell — WCL Outbound Dialer" — Manual trigger
- Configs: [`infra/n8n/retell-wcl-lead-intake-handler.json`](./infra/n8n/retell-wcl-lead-intake-handler.json), [`infra/n8n/retell-wcl-outbound-dialer.json`](./infra/n8n/retell-wcl-outbound-dialer.json)
- Both write to the **same Google Sheet** as VAPI (keyed on `vapi_call_id` column, which now stores Retell's `call_id`)

### Key Retell vs VAPI Payload Differences
| Field | VAPI path | Retell path |
|---|---|---|
| Event gate | `body.message.type === 'end-of-call-report'` | `body.event === 'call_ended'` |
| Intake data | `message.artifact.structuredOutputs[id].result` | `call.call_analysis.custom_analysis_data.intake_details` (JSON string) |
| Call direction | `metadata.source === 'outbound_dialer'` | `call.direction === 'outbound'` |
| Caller phone | `message.customer.number` | `call.from_number` (inbound) / `call.to_number` (outbound) |
| Duration | `message.durationSeconds` | `Math.round(call.duration_ms / 1000)` |
| Recording | `message.artifact.recordingUrl` | `call.recording_url` |
| Transcript | `message.artifact.transcript` | `call.transcript` |
| Call ID | `message.call.id` | `call.call_id` |
| End reason | `message.endedReason` | `call.disconnection_reason` |

### Retell Outbound Call API
```bash
# Fire an outbound call (once from_number is set)
curl -X POST "https://api.retellai.com/v2/create-phone-call" \
  -H "Authorization: Bearer key_f0db8002d57bbfb4f17db3b20a11" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "agent_8c36983cac060850dfb961406d",
    "from_number": "<RETELL_PHONE_NUMBER>",
    "to_number": "+14155551234",
    "retell_llm_dynamic_variables": {
      "dynamic_opening": "Hi John, Sergio Tapia here...",
      "first_name": "John", "last_name": "Doe", "phone": "+14155551234",
      "email": "john@example.com", "state": "CA", "property_type": "Single Family",
      "known_loan_type": "Fix & Flip", "transaction_type": "Purchase",
      "loan_amount_requested": "three hundred thousand dollars",
      "fico": "720", "inquiry_source": "Facebook", "inquiry_date": "April 22"
    },
    "metadata": {"source": "outbound_dialer", "sheet_row_number": "5"}
  }'
```

### Phone Number Porting (⚠️ Pending)
- **Number to port:** `+1 949-942-8395` (currently on VAPI)
- **Action needed:** Submit porting request in Retell Dashboard → Settings → Phone Numbers
- **Timeline:** 2–4 weeks typical
- **During transition:** VAPI number stays active; both systems run in parallel
- **After porting:** Set `from_number` in "Retell — WCL Outbound Dialer" node "Build Retell Payload" (search for `<RETELL_PHONE_NUMBER>`)

### Retell-specific Gotchas
- **No Liam voice built-in.** Using `11labs-Brian` as placeholder. Swap by PATCHing both agents with a new `voice_id`.
- **Sergio's first message is dynamic.** Passed via `retell_llm_dynamic_variables.dynamic_opening` per call. The LLM's `begin_message` is set to `{{dynamic_opening}}`.
- **Post-call analysis is async.** Retell fires a separate `call_analyzed` webhook after `call_ended`. If `call_analysis` fields are empty in `call_ended`, check if you need to handle `call_analyzed` event instead.
- **`intake_details` and `sergio_extras` are JSON strings** — always `JSON.parse()` them in the n8n Code node.

---

## Repo Layout

```
CLAUDE.md                             This file — context for Claude Code
SETUP.md                              How to deploy this system for a new client
README.md
infra/                                Live-config snapshot (source of truth on paper)
├── sync-infra.sh                     Pulls current state of n8n + VAPI + Retell into this folder
├── README.md
├── n8n/                              All 4 n8n workflows, live JSON (VAPI + Retell)
├── vapi/
│   ├── assistants/                   Alex + Sergio VAPI assistants
│   └── structured-outputs/           All 6 VAPI structured outputs
└── retell/
    ├── agents/                       Alex + Sergio Retell agents
    └── llms/                         Alex + Sergio Retell LLM configs
stage2-outbound/                      Stage 2 (outbound / Sergio persona)
├── README.md                         Architecture, persona, rules
├── STAGE2_SETUP.md                   Step-by-step setup guide
├── WCL_Lead_Pipeline_Template.csv    Google Sheet schema
└── outbound-first-message-template.md
docs/                                 Client-facing proposal + handoff docs
└── Proposal_Sergio_West_Capital_Lending.docx
tests/
```

---

## Suggested Next Steps

1. **Port +1 949-942-8395 to Retell.** Submit in Retell Dashboard. Once done, update `from_number` in the Retell outbound dialer.
2. **Test Retell inbound.** Assign Alex agent to a Retell phone number temporarily and make a test call. Verify email arrives and Sheet row is written.
3. **Professional Voice Clone of Sergio.** `11labs-Brian` is a placeholder; production should use Sergio's own ElevenLabs PVC, imported to Retell.
4. **Verify post-call analysis timing.** Check if Retell fires `intake_details` in `call_ended` or only in `call_analyzed` — may need to add a `call_analyzed` handler.
5. **CRM integration — Bonzo.** Add an HTTP node after Parse & Build Email that POSTs structured lead data to Bonzo.
6. **Hot-lead SMS/Slack alert.** Parallel branch off Parse & Build Email when `isHot = true`.
7. **Daily/weekly dialer schedule.** Swap the Retell outbound dialer's Manual Trigger for Cron.
8. **Stage 3 — ManyChat funnel.** Meta ad leads flow → ManyChat qualification → append to Google Sheet → dialer picks up automatically.
