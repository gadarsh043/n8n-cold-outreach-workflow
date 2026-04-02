# B2B Lead Generation (n8n Workflow)

This repo contains a single n8n workflow export: `B2B Lead Generation.json`.

At a high level, the workflow:
1. Pulls leads from Apollo based on a user-provided **Job Title** and **Location**.
2. Writes enriched leads into a Google Sheet as **Pending**.
3. Generates a personalized cold email (subject + body) with Groq.
4. Sends the email with Gmail and updates outreach status so you can continue the pipeline later.

## Triggers (two entry points)

### Trigger 1: Lead Scraper (form)
- Node: `Lead Generation Form` (`formTrigger`)
- User provides:
  - `Job Title`
  - `Location`
  - `Number of Leads`

Flow:
- `Apollo — Search Leads` searches Apollo.
- `Extract People Data` normalizes search results.
- `Apollo — Enrich Lead Data` enriches each lead (bulk match).
- `Format & Clean Lead Data` converts Apollo data into the sheet schema.
- `Save Leads to Sheet` writes/updates rows in your Google Sheet and sets `Outreach Status ` to `Pending`.
- Duplicate prevention is handled by `Skip Duplicates` + `Deduplicate Leads` + `removeDuplicates`.

### Trigger 2: Email Generator + Sender (manual)
- Node: `When clicking Execute Workflow` (`manualTrigger`)

Flow:
- Reads your Google Sheet (`Fetch Leads from Sheet`)
- Filters to leads that match:
  - `Outreach Status ` is `Pending`
  - `Email Address ` is not empty (`Filter Pending Leads`)
- Generates email content with:
  - `AI Cold Email Writer` (Groq via `Groq LLM (Fast AI)`)
- Sends email with:
  - `Send Cold Email via Gmail`
- Updates status with:
  - `Update the Outreach Status`

## Google Sheet schema (column headers must match exactly)

The workflow references column headers with exact spacing. In particular, several columns include a trailing space in the header name.

### Required columns

| Column header (exact) | Used for |
|---|---|
| `Apollo ID` | De-dup key (`matchingColumns`) |
| `Company` | Lead enrichment |
| `Job Title` | Lead enrichment + email context |
| `Full Name ` | Lead display (note trailing space) |
| `Phone Number` | Lead enrichment |
| `Profile URL ` | Lead enrichment (note trailing space) |
| `Email Address ` | Email recipient (note trailing space) |
| `Company Website` | Email context |
| `Industry Genre ` | Email context (note trailing space) |
| `Email Subject ` | Email subject |
| `Email Body` | Email body |
| `Outreach Status ` | Pipeline state (note trailing space) |

### Status values expected

- `Pending`: used by `Filter Pending Leads` to select leads that need email generation.
- `Mail Generated`: used by a filter node (`Filter1`) to select leads for the email sending loop.

Important: in the exported JSON, some “status update / fetch mail” nodes are missing column mappings (see **Configuration checklist**). You must ensure `Outreach Status ` is updated to `Mail Generated` at the appropriate time in your n8n UI.

## Credentials you must configure

The workflow requires:
- **Apollo API key** (header auth) for:
  - `https://api.apollo.io/api/v1/mixed_people/api_search`
  - `https://api.apollo.io/api/v1/people/bulk_match`
- **Google Sheets OAuth2**
- **Groq API key** (node: `Groq LLM (Fast AI)`)
- **Gmail OAuth2**

## Configuration checklist (nodes to verify in n8n)

The JSON export includes placeholders and/or incomplete parameters for several nodes. Before running, open the workflow in n8n and verify these nodes:

1. Spreadsheet URL + sheet name
   - Replace `YOUR_GOOGLE_SHEET_URL_HERE` everywhere it appears.
   - Ensure the correct sheet name is set to `Master Sheet` (the workflow uses `Master Sheet` for lead saving).

2. Email persistence + status update (may be blank in export)
   - `Save Email to Sheet`
   - `Fetch Leads for Mail`
   - `Update the Outreach Status`
   - `Fetch Existing Lead IDs`

   In the exported file, these nodes show minimal parameters / empty `sheetName` in several places.
   In the n8n UI, configure them so that:
   - `Email Subject ` and `Email Body` are written to the sheet from the AI output (`AI Cold Email Writer` outputs JSON with `Email Subject` and `Email Body`).
   - `Outreach Status ` is set to `Mail Generated` when you want the “send” loop to run.
   - The “fetch” nodes read the right rows based on `Outreach Status `.

3. Gmail sender details
   - Node: `Send Cold Email via Gmail`
   - Update `senderName` (currently `Your Name`) to your actual name/brand.

## Customizing the AI cold email prompt

Node: `AI Cold Email Writer`

It instructs the model to:
- Write as a B2B cold email copywriter for `[COMPANY NAME]`
- Never use: `I hope this email finds you well`
- Never mention pricing
- Use at most 4 paragraphs
- Output a **raw JSON object only** with exactly:
  - `Email Subject`
  - `Email Body`

Update this prompt’s company-specific section:
- Replace the `[COMPANY NAME]` placeholders and the bullet points under `ABOUT [COMPANY NAME]` with your actual:
  - services
  - unique value proposition
  - website URL
  - contact number

You can also swap the Groq model in `Groq LLM (Fast AI)` (currently `qwen/qwen3-32b`).

## How to run

### Step 1: Generate leads
1. Open n8n.
2. Find the workflow `B2B Lead Generation`.
3. Execute Trigger 1 by submitting `Lead Generation Form` with:
   - a job title
   - a location
   - a lead count
4. Confirm your Google Sheet rows are created/updated with:
   - outreach status `Pending`
   - populated `Email Address ` (filtering will skip blank emails)

### Step 2: Generate and send emails
1. Run the manual trigger: `When clicking Execute Workflow`.
2. Confirm that:
   - leads in `Pending` are receiving AI-generated `Email Subject ` + `Email Body`
   - your “send” loop finds leads with `Outreach Status ` = `Mail Generated`
   - status updates occur after the Gmail send

If the email sending loop does nothing, the most common cause is that `Outreach Status ` never becomes `Mail Generated` due to the missing column mappings described above.

## Throttling / cooldown behavior

The workflow includes explicit wait nodes to reduce rate-limits and smooth the execution:
- `Wait — Apollo Cooldown (2s)`
- `Wait — Email Cooldown (4s)`
- `Wait — Email Cooldown (60s)`

## Known quirks to be aware of (from the export)

- Several nodes in the email/sending portion are missing exported column mappings (notably `Save Email to Sheet`, `Fetch Leads for Mail`, `Update the Outreach Status`, `Fetch Existing Lead IDs`). Treat them as “configure in n8n” nodes.
- The Apollo enrichment bulk match references a single ID (`$('Deduplicate Leads').first().json.id`). If you intend to enrich multiple leads per run, you may need to adjust that node logic.

## License

No license file is included in this export. Add one if you plan to publish or share this workflow.

