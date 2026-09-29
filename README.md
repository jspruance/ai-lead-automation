# AI Lead Automation — Speed-to-Lead

A speed-to-lead workflow built in [n8n](https://n8n.io) for a commercial cleaning business. It takes an inbound web inquiry, validates it, uses AI to qualify it, routes it, updates HubSpot, and sends personalized responses and follow-ups within seconds.

> **Portfolio project.** This is a use-case implementation for a fictional commercial cleaning company ("Cascade Commercial Cleaning", Seattle area). It is not paid client work. All test leads are fictional.

## How it works

```
Webhook / Test trigger
  → Normalize & validate lead
  → AI lead analysis (structured JSON)
  → Validate AI output
  → Deterministic qualification rules
  → Route: HIGH_PRIORITY | NOT_QUALIFIED | HUMAN_REVIEW
```

**1. Lead intake:** a `POST /speed-to-lead` webhook receives form submissions (first name, last name, email, phone, company, message). A manual **Test Scenario Trigger** can inject one of three fictional test leads instead.

**2. Normalization and validation:** trims and normalizes the fields, lowercases the email, and strips the phone number down to digits. A lead needs a well-formed email and a message; invalid leads go no further.

**3. AI analysis:** OpenAI (`gpt-5-mini`) extracts structured facts only: customer type, service, location, facility count, intent, timeline, recurring service, confidence, and a reasoning summary. The prompt tells the model not to make the routing decision itself.

**4. Output validation:** the model's JSON is parsed and checked against the allowed values and types. A malformed response is flagged, not trusted.

**5. Qualification rules:** deterministic code, not the LLM, makes the decision:

- Failed AI validation, or confidence below 0.75 → `HUMAN_REVIEW`
- A residential customer or service → `NOT_QUALIFIED`
- A commercial cleaning quote request with a timeline within 90 days → `HIGH_PRIORITY`
- Anything else → `HUMAN_REVIEW`

**6. Routing:**

- **HIGH_PRIORITY:**
  - **HubSpot:** creates or updates the contact by email and looks up the company by email domain, creating the company only if none is found (this avoids duplicate CRM records). It then creates a deal and associates it with the contact and company through the HubSpot v4 associations API.
  - **Response:** generates a personalized reply with AI (the prompt forbids invented pricing and availability) and sends it.
  - **Sales alert:** emails an internal notification with the lead details, the AI analysis, and the HubSpot deal.
  - **Follow-up:** after a wait, checks Gmail for a reply from the lead. If there is none, it generates and sends one AI-written follow-up.
- **NOT_QUALIFIED:** sends a polite AI-generated "not a fit" reply.
- **HUMAN_REVIEW:** emails a reviewer the lead, the AI interpretation, the confidence, and the reason for review. No automated sales response is sent.

## Stack

n8n · OpenAI (`gpt-5-mini`) · HubSpot CRM (n8n node + v4 associations API) · Gmail

## Using this workflow

1. Import `workflows/speed-to-lead-commercial-cleaning.json` into n8n.
2. Create OpenAI, HubSpot, and Gmail credentials and assign them to the matching nodes (credentials are not included).
3. Replace these placeholders:

| Placeholder | Where |
| --- | --- |
| `YOUR_HUBSPOT_PRIVATE_APP_TOKEN` | "Associate Deal to Contact/Company" HTTP headers. It's better to switch these nodes to an n8n HubSpot credential than to hardcode a token. |
| `REPLACE_WITH_HUBSPOT_DEAL_STAGE_ID` | "Create Deal" stage |
| `demo-inbox@example.com` | All Gmail send nodes. In this demo, every outbound email (including the replies to leads) goes to one internal inbox. For production, point the lead-facing nodes at the lead's email address. |

4. The follow-up wait is 2 minutes for demo purposes; lengthen it for real use.

## Repository layout

| Path | Contents |
| --- | --- |
| `workflows/` | Sanitized n8n workflow export |
| `docs/` | Additional documentation |
| `sample-data/` | Reserved for synthetic sample inputs |
