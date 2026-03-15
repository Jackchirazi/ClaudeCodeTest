# Lead Gen System — Setup Guide

Step-by-step instructions to get the full system running. Follow phases in order. Phase 1 gets you a working manual qualification pipeline; later phases add automation.

---

## Prerequisites — Accounts to Create

Sign up for these before starting. All have free tiers.

| Service | URL | What It's For |
|---|---|---|
| n8n Cloud | n8n.io | Automation engine (all workflows run here) |
| Airtable | airtable.com | Database + dashboard |
| Anthropic (Claude API) | console.anthropic.com | AI qualification + outreach generation |
| Hunter.io | hunter.io | Email discovery |
| Browserless.io | browserless.io | JavaScript-heavy website scraping (Phase 2) |
| SerpAPI | serpapi.com | Google search automation (Phase 3) |

---

## Phase 1: Foundation

### Step 1 — Set Up Airtable

1. Create a new Airtable Base called **Lead Gen System**
2. Create these tables with exactly these field names:

**Table: Brands**
| Field | Type |
|---|---|
| brand_name | Single line text |
| category | Single select: fragrance / cosmetics / clothing |
| approximate_msrp | Currency |
| your_price | Currency |
| availability_status | Single select: in_stock / limited / out_of_stock |
| notes | Long text |

**Table: Leads**
| Field | Type |
|---|---|
| retailer_name | Single line text |
| website_url | URL |
| lead_score | Number (0-10) |
| recommendation | Single select: hot_lead / worth_pursuing / maybe_later / skip |
| qualification_json | Long text |
| status | Single select: qualified / contacts_found / outreach_drafted / contacted / responded / converted / dead |
| contacts_count | Number |
| analysis_date | Date |

**Table: Contacts**
| Field | Type |
|---|---|
| lead_id | Single line text |
| retailer_name | Single line text |
| first_name | Single line text |
| last_name | Single line text |
| email | Email |
| title | Single line text |
| confidence | Number |
| source | Single line text |

**Table: Outreach**
| Field | Type |
|---|---|
| lead_id | Single line text |
| channel | Single select: email / linkedin / call_script |
| subject | Single line text |
| message | Long text |
| talking_points | Long text |
| personalization_notes | Long text |
| status | Single select: draft / approved / sent / replied |
| generated_at | Date |

3. Import the brand catalog: Go to **Brands** table → Import → CSV → upload `config/brands.csv`

4. Get your **Airtable API Key and Base ID:**
   - API Key: airtable.com → Account → Developer Hub → Personal access tokens → Create token (scopes: data.records:read, data.records:write)
   - Base ID: Open your base in browser → URL will show `airtable.com/appXXXXXXXXXXXXXX` — that `appXXX...` part is your Base ID

---

### Step 2 — Set Up n8n Cloud

1. Sign up at n8n.io → Start a cloud instance
2. In n8n, go to **Settings → Environment Variables** and add:

```
AIRTABLE_BASE_ID     = appXXXXXXXXXXXXXX     (from Step 1)
ANTHROPIC_API_KEY    = sk-ant-XXXXXXXXXX       (from console.anthropic.com)
HUNTER_API_KEY       = XXXXXXXXXXXXXXXX        (from hunter.io → API)
N8N_WEBHOOK_BASE     = https://your-n8n-instance.app.n8n.cloud  (your n8n URL, no trailing slash)
SERPAPI_KEY          = XXXXXXXXXXXXXXXX        (Phase 3 only)
```

3. Set up **Airtable credentials** in n8n:
   - Go to Credentials → New → Airtable Token API
   - Paste your Airtable personal access token
   - Name it "Airtable Lead Gen"

---

### Step 3 — Import n8n Workflows

Import workflows in order:

1. In n8n, click **Workflows → Import from File**
2. Import `n8n/01-manual-qualify-lead.json`
3. Open the workflow, click each node that says "Airtable" and connect it to your Airtable credentials
4. Click **Activate** (toggle top-right)
5. Copy the **Webhook URL** from the "Webhook: Qualify URL" node — you'll need this

Repeat for:
- `n8n/02-contact-finder.json` (activate after 01)
- `n8n/03-outreach-generator.json` (activate after 02)

**Don't import 04 and 05 yet** — those are Phase 3.

---

### Step 4 — Test the Pipeline

Test with a URL you already know is a good lead:

1. Use a tool like Postman, Insomnia, or just your browser's URL bar to send:
   ```
   POST https://your-n8n.app.n8n.cloud/webhook/qualify
   Body: {"url": "https://example-retailer.com"}
   ```

   Or use curl:
   ```bash
   curl -X POST https://your-n8n.app.n8n.cloud/webhook/qualify \
     -H "Content-Type: application/json" \
     -d '{"url": "https://example-retailer.com"}'
   ```

2. Check the n8n execution log — should show each node running green
3. Check Airtable Leads table — new record should appear with qualification JSON and lead score
4. If score ≥ 6, check Contacts table — should have contacts from Hunter.io

**Run 5 test URLs** you've manually qualified before. Compare the AI scores to your own judgment to calibrate.

---

## Phase 2: Contact Discovery + Outreach

### Step 5 — Set Up Hunter.io

1. Sign up at hunter.io (free tier: 25 searches/month)
2. Get API key: Hunter.io → Account → API
3. Add to n8n environment variables: `HUNTER_API_KEY`
4. The contact finder workflow (02) is already configured for Hunter.io — just activate it

### Step 6 — Generate Outreach for a Lead

1. Find a lead in Airtable with score ≥ 7
2. Copy its Airtable record ID (shown in the URL when you open the record: `recXXXXXXXXXXXXXX`)
3. Trigger the outreach generator:
   ```bash
   curl -X POST https://your-n8n.app.n8n.cloud/webhook/generate-outreach \
     -H "Content-Type: application/json" \
     -d '{"lead_id": "recXXXXXXXXXXXXXX", "channel": "all"}'
   ```
4. Check the Outreach table in Airtable — should have email, LinkedIn, and call script variants
5. Review and edit the messages before using them

---

## Phase 3: Chatbot + Automation

### Step 7 — Import and Activate Chatbot

1. Import `n8n/04-chatbot-agent.json`
2. Activate it
3. Note the **Webhook URL** for the chat endpoint
4. Test it:
   ```bash
   curl -X POST https://your-n8n.app.n8n.cloud/webhook/chat \
     -H "Content-Type: application/json" \
     -d '{"message": "https://example-retailer.com"}'
   ```

The chatbot understands:
- `"qualify https://example.com"` — runs full pipeline
- `"show me leads scored 8+"` — returns top leads from Airtable
- `"do we carry Tom Ford?"` — checks catalog
- `"write an email for [store name]"` — triggers outreach generation

### Step 8 — Set Up Automated Discovery (Optional)

1. Sign up for SerpAPI (100 free searches/month)
2. Add `SERPAPI_KEY` to n8n environment variables
3. Import `n8n/05-lead-discovery.json`
4. The workflow runs **daily at 8 AM** by default
5. It will automatically find new retailer URLs and feed them into the qualification pipeline

Adjust the search queries in the "Generate Search Queries" node to match the brands you're currently pushing.

---

## Phase 4: Airtable Dashboard

### Set Up Lead Pipeline View

In Airtable, create these views in the Leads table:

1. **Pipeline** — Group by `status`, sort by `lead_score` descending
2. **Hot Leads** — Filter: `lead_score >= 8`, sort by score
3. **To Contact** — Filter: `status = contacts_found`, sort by score
4. **In Progress** — Filter: `status IN (contacted, responded)`
5. **Won** — Filter: `status = converted`

### Airtable Interface (Optional)

1. Go to Interfaces → Build Interface
2. Create a record review interface for the Leads table
3. Add buttons to manually trigger outreach or update status

---

## API Keys Reference

| Variable | Where to Get It |
|---|---|
| `AIRTABLE_BASE_ID` | Airtable URL: `airtable.com/appXXXX...` → the `appXXXX` part |
| `ANTHROPIC_API_KEY` | console.anthropic.com → API Keys |
| `HUNTER_API_KEY` | hunter.io → Account → API |
| `SERPAPI_KEY` | serpapi.com → Dashboard → API Key |
| `N8N_WEBHOOK_BASE` | Your n8n cloud URL, no trailing slash |

---

## Troubleshooting

**Workflow not triggering:**
- Check the webhook is activated (green toggle in top-right of workflow)
- Confirm you're using the correct webhook URL (copy from the webhook node, not the workflow URL)

**AI giving poor qualification results:**
- Provide example qualified sites in the qualification prompt (`prompts/qualification.md`) using few-shot examples
- Adjust the scoring weights in `config/scoring-weights.json` to match your judgment

**Hunter.io finding no contacts:**
- Some sites have no email footprint — try LinkedIn manually for these
- Hunter.io works best with company domains, not marketplace sellers

**Airtable storage failing:**
- Check credentials are connected in each node
- Verify the Base ID is correct (common mistake: copying workspace ID instead)
- Ensure all field names in n8n match exactly what's in Airtable

---

## Monthly Cost Summary

| Tool | Plan | Cost |
|---|---|---|
| n8n Cloud | Starter | ~$24/mo |
| Airtable | Plus | ~$12/mo |
| Claude API (Sonnet) | Pay per use | ~$10-30/mo |
| Browserless.io | Starter (Phase 2) | ~$10/mo |
| Hunter.io | Free tier | $0 (25/mo) |
| SerpAPI | Free tier | $0 (100/mo) |
| **Total** | | **~$56-76/mo** |
