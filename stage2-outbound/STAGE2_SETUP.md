# Stage 2 — Outbound Dialer Setup

## What's Built

### Google Sheet: "WCL Lead Pipeline"
- **URL**: https://docs.google.com/spreadsheets/d/1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc/edit
- **Sheet ID**: `1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc`
- 48 columns covering lead identity, call metadata, loan classification, DSCR/Fix-Flip fields, call outcome, retry tracking, CRM sync
- 3 mock leads pre-populated (John Martinez / Fix & Flip, Sarah Chen / DSCR, Marcus Williams / blank)

### n8n Workflow 1: "Vapi — WCL Outbound Dialer" (already deployed)
- **ID**: `e5OaddLYhhV0aS2Z`
- **URL**: https://buildflows.cloud/workflow/e5OaddLYhhV0aS2Z
- Flow: Manual Trigger → Read Sheet → Filter Ready → Split In Batches (size=1) → Build VAPI Payload → VAPI Start Call → Mark Row In Progress → Pause → loop
- Uses VAPI number `+19499428395` (phoneNumberId `905a5925-e228-4144-bcac-3e4fa81f58c4`)
- Uses existing Alex assistant with outbound-tuned `firstMessage` override (3 templates: unknown / DSCR / Fix & Flip)

### n8n Workflow 2: "Vapi — WCL Lead Intake Handler" (existing; needs manual update)
- **ID**: `MDXGW2s91yXtjyT7`
- **Pending update saved at**: `inbound-workflow-with-sheets-sync.json`
- When applied, it will:
  - Detect outbound vs inbound calls via `call.assistantOverrides.metadata.source`
  - Write/upsert every call result to the Google Sheet
  - Keep the existing Resend email alert

---

## One-Time Setup (~5 minutes)

### Step 1 — Create Google Sheets OAuth Credential in n8n

1. Log into n8n: https://buildflows.cloud
2. Top-right menu → **Credentials** → **+ Add credential**
3. Search for **"Google Sheets OAuth2 API"** → select it
4. Click **Sign in with Google** — authorize with `chris@gobehindtheproduct.com`
5. Save. Note the credential name (e.g., "Google Sheets account")

### Step 2 — Wire the Credential Into Both Workflows

Open **Vapi — WCL Outbound Dialer** (`e5OaddLYhhV0aS2Z`):
- Click `Read Lead Pipeline` node → Credential dropdown → select the Google Sheets credential you just made
- Click `Mark Row In Progress` node → same thing
- Click `VAPI Start Call` node → Credential dropdown → add an **HTTP Header Auth** credential named "VAPI API Key":
  - Header Name: `Authorization`
  - Header Value: `Bearer a6b34170-e0ef-432a-af14-bff69dd08b6e`
- Save the workflow

### Step 3 — Update the Inbound Workflow

The inbound workflow needs sheet-sync added. Two options:

**Option A — Import the updated JSON (recommended)**
1. Open **Vapi — WCL Lead Intake Handler** (`MDXGW2s91yXtjyT7`)
2. Click the ⋮ menu → **Import from File** → select `inbound-workflow-with-sheets-sync.json` from this folder
3. (Overwrite the existing workflow)
4. Click each Google Sheets node → assign the Google Sheets credential
5. Activate

**Option B — Edit manually**
1. Open the inbound workflow
2. After `Parse & Build Email`, add a new **Code** node called "Flatten sheetRow":
   ```javascript
   return [{ json: $input.first().json.sheetRow }];
   ```
3. After that, add a **Google Sheets** node "Upsert to Lead Pipeline":
   - Operation: `Append or Update Row`
   - Document ID: `1O9ZV7gG_ZhWPX558ZW4ga1hmvl4hcbpZ-ZAvxfwJAyc`
   - Sheet: `Sheet1` (gid 161758208)
   - Mapping: Auto-map input data
   - Match on: `vapi_call_id`
4. Wire: `Parse & Build Email → Flatten sheetRow → Upsert to Lead Pipeline` (parallel to existing Send via Resend)
5. Replace the `Parse & Build Email` code with the version in the updated JSON (adds `sheetRow` field to output)

---

## How to Demo

### Demo Flow A — Outbound Dialing
1. Open the **Google Sheet**
2. Add a new row with your own phone number as a test lead
3. Set `call_status = Pending`
4. Go to n8n → **Vapi — WCL Outbound Dialer** → click **Execute Workflow**
5. Your phone rings within ~5 seconds
6. Alex greets you with the outbound opening ("Hi [Name], this is Alex calling from West Capital Lending...")
7. Answer a few qualification questions, then hang up
8. ~15 seconds after the call ends, the row updates with transcript URL, qualification summary, all captured fields, and Resend fires the lead email

### Demo Flow B — Inbound (still works as before)
1. Call `+1 949-942-8395`
2. Run through the DSCR or Fix & Flip script
3. After hangup: email arrives AND a new row appears in the Google Sheet

---

## Changing Config Later

| Want to… | Edit this |
|---|---|
| Enable batch calling (e.g., 5 at a time) | `Split In Batches` node → `batchSize` |
| Schedule daily dialing instead of manual | Replace `Manual Trigger` with a `Cron` node (e.g., daily at 9am) |
| Tighten retry window | `Filter Ready to Dial` code node → change `MAX_RETRIES` or retry time calc |
| Tune outbound opening | `Build VAPI Payload` code node → edit the three `firstMessage` templates |
| Swap Google Sheets for Airtable / CSV / API | Replace `Read Lead Pipeline` node with a new data source; the rest stays identical |

---

## Files in This Folder

| File | Purpose |
|---|---|
| `WCL_Lead_Pipeline_Template.csv` | Original schema (already imported to Google Sheet) |
| `outbound-dialer-workflow.json` | Backup of deployed Outbound Dialer |
| `inbound-workflow-with-sheets-sync.json` | Updated inbound workflow (to be imported manually) |
| `outbound-first-message-template.md` | The three firstMessage templates Alex uses on outbound calls |
| `STAGE2_SETUP.md` | This file |
