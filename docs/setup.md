# Setup Guide — RB2B → HubSpot De-anonymization

## Prerequisites

- n8n instance (self-hosted or cloud) running v1.0+
- HubSpot account with API access (Marketing Hub Pro or Sales Hub Pro+)
- RB2B account with the website pixel installed
- Clay account (or alternative enrichment provider)
- Zapier account for last-mile notifications

## Step 1: Import the workflow

1. In n8n, **Workflows → Import from File**
2. Select `workflow.json` from this repo
3. The workflow imports as "RB2B - Deanonymization workflow"

## Step 2: Wire credentials

### HubSpot

1. Click any HubSpot node (e.g., `Contact Search`)
2. **Credential → Create New**
3. Authenticate via HubSpot OAuth — make sure the connecting account has these scopes:
   - `crm.objects.contacts.read` / `write`
   - `crm.objects.companies.read` / `write`
   - `crm.objects.deals.read`
   - `crm.objects.engagements.read` / `write`
4. Apply the credential to all 22 HubSpot nodes (n8n will offer to do this in bulk)

### Clay

1. In your Clay workspace, create a new table or open an existing one
2. **Add Source → Webhook** → copy the webhook URL
3. In n8n, open the `Clay` HTTP request node
4. Replace the URL with your Clay webhook URL
5. The workflow sends company domain + visitor metadata; configure your Clay table to enrich on those fields

### Zapier

1. In Zapier, create two new Zaps:
   - **Zap 1: AE notification on hot signal** → Webhooks (Catch) → Slack (DM the AE)
   - **Zap 2: Reactivation trigger** → Webhooks (Catch) → your reactivation tool of choice
2. Copy the catch hook URLs from each Zap
3. In n8n, find the two HTTP request nodes with `{{ZAPIER_WEBHOOK_URL}}` placeholders and replace them with your real URLs

## Step 3: Wire RB2B → webhook

1. Activate the workflow in n8n
2. Open the `Webhook` node and copy the **Production URL** (looks like `https://your-n8n.com/webhook/rb2b-lead`)
3. In RB2B → **Integrations → Webhook** → paste the URL
4. Save and test from RB2B's interface

## Step 4: Configure routing logic

The workflow's routing depends on your HubSpot deal stages. Open these nodes and update the stage IDs:

- **Meeting Status (switch)** — main stage router
- **Meeting Status1 / Meeting Status2** — secondary routers for different lead types

The default expects stages like `appointmentscheduled`, `qualifiedtobuy`, `presentationscheduled`, `closedwon`, `closedlost`. Map to your pipeline's actual stage IDs (find them via `GET /crm/v3/pipelines/deals` in HubSpot's API).

## Step 5: Test

1. From RB2B's dashboard, trigger a test webhook
2. Watch the n8n execution — it should route through the correct branch based on whether the test contact exists in HubSpot
3. Verify in HubSpot that the engagement was created on the correct object (contact or deal)

## Troubleshooting

**Every RB2B event creates a new contact**
→ HubSpot search is failing to find existing matches. Check that the `Contact Search` node is searching by `email` AND that the RB2B payload includes a valid email. If RB2B only provides LinkedIn URLs, switch the search property accordingly.

**Engagements aren't appearing in HubSpot timelines**
→ Verify the engagement type (`NOTE` / `MEETING` / `CALL`) matches what your team uses for filtering. Check that the association ID points to the right object type.

**Duplicate engagements on retry**
→ The webhook is receiving the same payload twice. Confirm RB2B isn't configured to retry on every response; the workflow's dedup catches duplicate contacts but engagement creation is intentionally not deduped (different visits = different engagements).

**Clay enrichment is slow / blocking the workflow**
→ Clay enrichment runs synchronously by default. To make it async, change the `Clay` node's mode to fire-and-forget and have Clay write back to HubSpot directly rather than continuing the n8n flow.
