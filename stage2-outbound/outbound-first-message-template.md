# Outbound First Message Templates (VAPI assistantOverrides.firstMessage)

The outbound flow injects the lead's first name and (optionally) known loan type into the opening line. After the opening, Alex's existing system prompt handles the full qualification flow unchanged.

## Template A — Unknown loan type (cold call, we only have name/phone)
```
Hi {{first_name}}, this is Alex calling from West Capital Lending. Do you have a quick minute? I'm reaching out because we help real estate investors like yourself get financed on rental properties and fix and flip projects... and I'd love to find out what you're working on so we can point you in the right direction.
```

## Template B — Known DSCR lead (prior intake or lead source tagged it)
```
Hi {{first_name}}, this is Alex calling from West Capital Lending about a rental property financing inquiry. Do you have a quick minute to walk through a few details so we can see what you qualify for?
```

## Template C — Known Fix & Flip lead
```
Hi {{first_name}}, this is Alex calling from West Capital Lending about a fix and flip project you're working on. Do you have a quick minute to go over a few details so we can line up the right loan for you?
```

## How the n8n workflow picks the template
In the workflow's JavaScript node (before the VAPI call):
- If `loan_type` column is empty → Template A
- If `loan_type == "DSCR"` → Template B
- If `loan_type == "Fix & Flip"` → Template C

## VAPI API payload shape
```json
POST https://api.vapi.ai/call/phone
{
  "assistantId": "12b07537-547c-4fb5-bb5a-8a7ab0c1759d",
  "phoneNumberId": "<TBD — user supplies>",
  "customer": {
    "number": "+14155551234",
    "name": "John Martinez"
  },
  "assistantOverrides": {
    "firstMessage": "Hi John, this is Alex calling from West Capital Lending...",
    "variableValues": {
      "first_name": "John",
      "last_name": "Martinez",
      "known_loan_type": "Fix & Flip"
    },
    "metadata": {
      "source": "outbound_dialer",
      "sheet_row_id": "3"
    }
  }
}
```

The `metadata.sheet_row_id` lets the end-of-call webhook find the correct row to update.
