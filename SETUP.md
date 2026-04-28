# Setup Instructions for New Client Deployment

Use this guide to quickly replicate the system for a new client (different lender, intake flow, loan products, etc.).

## Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/real-estate-lead-gen-system.git your-client-name
cd your-client-name
```

## Step 2: Update CLAUDE.md with New Client Credentials

Open `CLAUDE.md` and update:

### VAPI Section
```
- **Assistant ID:** [Get from VAPI dashboard after creating new assistant]
- **VAPI Private API Key:** [From VAPI API keys page]
- **TTS Voice:** [Pick from VAPI voices — recommend ElevenLabs Flash v2.5]
```

### System Prompt
Edit the prompt to match the new client's:
- Company name and branding
- Loan products (replace DSCR/Fix & Flip with your products)
- Intake questions and flow
- Formatting rules if different
- Call opening/closing messages

### n8n Section
```
- **n8n URL:** [Your n8n instance — can be same VPS or different]
- **REST API Key:** [From n8n API key settings]
- **Workflow ID:** [After creating workflow in n8n]
```

### Resend Section
```
- **API Key:** [New Resend API key for this client's account]
- **Verified Domain:** [Client's domain, e.g., newlender.com]
- **Sender:** [leads@newlender.com]
- **Recipient:** [Client's email]
```

## Step 3: Create VAPI Agent

1. Log into VAPI dashboard
2. Click "Create Assistant"
3. Name it (e.g., "Alex — New Lender Inc")
4. Configure:
   - **LLM:** Claude Sonnet 4.6 (or GPT-4o-mini for lower cost)
   - **STT:** Deepgram Nova-2
   - **TTS:** ElevenLabs, model `eleven_flash_v2_5`, pick a voice
   - **Temperature:** 0.4
5. Paste the system prompt from CLAUDE.md into the "System Prompt" field
6. Set **Server URL** to your n8n webhook path (ask if unsure)
7. Set **Server Message Type** to `end-of-call-report`
8. Copy the **Assistant ID** and paste it into CLAUDE.md
9. Save

## Step 4: Create n8n Workflow

1. Log into your n8n instance
2. Create a new workflow
3. Add three nodes:
   - **Webhook (VAPI)** — incoming POST from VAPI
   - **Code (JS)** — parse payload and build email (copy the code from your reference docs or prior deployment)
   - **HTTP Request** — POST to Resend API
4. Configure the Webhook:
   - Path: `/webhook/your-client-webhook-path`
   - Method: POST
5. Configure the Code node:
   - Update the email recipient, sender, subject format for new client
   - Update any client-specific fields (colors, logo, terminology)
6. Configure the HTTP Request:
   - URL: `https://api.resend.com/emails`
   - Method: POST
   - Headers: `Authorization: Bearer {RESEND_API_KEY}`
   - Body: Pass the email object from the Code node
7. Test with `bash tests/webhook-test.sh` (after updating the URL)
8. Copy the **Workflow ID** and paste it into CLAUDE.md

## Step 5: Set Up Resend

1. Create a Resend account (or use existing one)
2. Add a new domain (e.g., newlender.com)
3. Add the DNS records shown in Resend to your DNS provider
4. Verify the domain in Resend
5. Copy the **API Key** and paste it into CLAUDE.md (and n8n HTTP node)

## Step 6: Update README.md

Replace client-specific references:
- "West Capital Lending" → "New Lender Inc"
- "DSCR and Fix & Flip loans" → your loan types
- "chris@gobehindtheproduct.com" → client's email
- "leads@gobehindtheproduct.com" → client's sender domain
- Example property types, credit ranges, etc.

## Step 7: Test

### Webhook Test (No Phone Call)
```bash
# Update the webhook URL in tests/webhook-test.sh to point to your new n8n instance
bash tests/webhook-test.sh

# Should return a Resend email ID if successful
```

### Live Call Test
1. Get the VAPI phone number from the VAPI dashboard
2. Call it
3. Follow the sample answers in `tests/dscr-sample-answers.txt` (or create new ones for your products)
4. Hang up and wait 15-30 seconds
5. Check email at the recipient address

## Step 8: Push to GitHub

```bash
git remote add origin https://github.com/yourusername/real-estate-lead-gen-system.git
git branch -M main
git push -u origin main
```

## Customization Examples

### Change the Voice
In VAPI, update `voice.voiceId` to a different ElevenLabs ID. Popular options:
- `TX3LPaxmHKxFdv7VOQHJ` — Liam (friendly American male)
- `ErXwobaYiN019PkySvjV` — Antoni (warm, professional)
- `VR6AewLTigWG4xSOukaG` — Arnold (authoritative)

### Change Email HTML
Edit the `html` variable in the n8n Code node to match client branding:
- Update colors (currently navy/gold)
- Add logo
- Adjust table layout
- Change terminology

### Add CRM Integration
In n8n, add a new HTTP node after "Send via Resend":
- URL: `https://api.example-crm.com/leads`
- Headers: API key for the CRM
- Body: Extract lead data from the parsed payload
- Only run if email was sent successfully

### Change Intake Flow
Edit the system prompt in CLAUDE.md and VAPI:
- Add/remove questions
- Reorder sections
- Change property types, loan products, credit ranges
- Update terminology for industry

---

**Questions?** Refer to README.md for full documentation and troubleshooting.
