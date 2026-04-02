# B2B Lead Generation (n8n)

This project is a single n8n workflow export: `B2B Lead Generation.json`.

It automates a simple outbound loop:
1. Get leads from Hunter.io using a domain (from a form).
2. Save leads into Google Sheets with `Outreach Status = Pending`.
3. Generate a personalized cold email for pending leads using Groq.
4. Send the email via Gmail and update outreach status in your sheet (once you configure the status transition in the workflow).

> Note: The workflow export still contains some Apollo-related/unconnected nodes from earlier iterations. The active flow for lead scraping uses Hunter.io (see `Lead Generation Form` → `Search Leads`).

## Workflow overview

### Trigger 1: Lead scrape (form)
Entry point:
- `Lead Generation Form` (`formTrigger`)

Connected path:
- `Lead Generation Form`
  -> `Search Leads` (HTTP request to Hunter.io)
  -> `Code in JavaScript` (filters Hunter results to “technical” people)
  -> `Extract People Data` (normalizes Hunter objects into rows)
  -> `Save Leads to Sheet` (append/update by `ID`)

### Trigger 2: Email generator + sender (manual)
Entry point:
- `When clicking Execute Workflow` (`manualTrigger`)

Connected path:
- `Fetch Leads from Sheet`
  -> `Filter Pending Leads` (needs an email + `Outreach Status = Pending`)
  -> `AI Cold Email Writer` (Groq; returns `Email Subject` + `Email Body`)
  -> `Save Email to Sheet` (writes subject/body into the sheet)
  -> `Wait — Email Cooldown (4s)`
  -> `Fetch Leads for Mail`
  -> `Remove Duplicates`
  -> `Filter1` (selects rows where `Outreach Status = Mail Generated`)
  -> `Loop Over Items`
  -> `Send Cold Email via Gmail`
  -> `Update the Outreach Status`
  -> `Wait — Email Cooldown (60s)` and continues the loop

## Google Sheets setup (required)

The workflow is configured to write to:
- Sheet document id: `1cGZWNhzSSani12XS9wvPwMwd_4VPmRgV3TY4FFs5ZTA`
- Sheet tab name: `Master Lead Generation sheet`

You must ensure your sheet has column headers that match what the workflow references.

### Columns written/used by the workflow

These appear in the workflow’s `Save Leads to Sheet` node and `Extract People Data` logic:
- `ID`
- `Full Name`
- `Job Title`
- `Company`
- `Email Address`
- `Phone Number`
- `Profile URL`
- `Company Website`
- `Industry Genre`
- `Outreach Status`
- `Email Subject`
- `Email Body`

These headers are also referenced elsewhere in the workflow (pay special attention to whitespace; see below).

### Important: header whitespace mismatch checks

In the exported JSON, some nodes reference headers with different whitespace than others:
- `Filter Pending Leads` uses: `Email Address` and `Outreach Status` (no trailing whitespace)
- `Send Cold Email via Gmail` uses: `Email Address ` (with a trailing space)
- `Filter1` uses: `Outreach Status ` (with a trailing space)

If your sheet headers don’t include trailing spaces, the email-sending/filtering portion may do nothing.

In n8n, open these nodes and make them consistent:
- `Send Cold Email via Gmail`: set `sendTo` to `...['Email Address']`
- `Filter1`: set `leftValue` to `...['Outreach Status']`

Or, alternatively, edit your Google Sheet headers to include the trailing spaces (not recommended).

## Hunter.io setup

The workflow uses Hunter’s Domain Search endpoint:
- `POST https://api.hunter.io/v2/domain-search`

The `Search Leads` node passes these query parameters:
- `domain` = from `Domain Name` in the form
- `seniority` = `executive, senior`
- `api_key` = set in the node (currently hardcoded in the export; you should replace it)

### Credentials you should configure in n8n

Best practice: don’t hardcode your API key in the node JSON.

In n8n:
1. Open `Search Leads` (HTTP Request).
2. Replace the `api_key` query parameter value with either:
   - an n8n credential (if you configured one for Hunter), or
   - an environment variable, or
   - an n8n expression pointing to a secret you manage.

If you keep it as-is initially, you can still run, but you should update it before committing/redistributing.

### Lead filtering logic (important)

After Hunter returns email candidates, the workflow filters them in:
- `Code in JavaScript`:
  - Keeps people when their `position` or `department` matches “technical” keywords (e.g., engineer, engineering, CTO, backend, frontend, developer, devops, infrastructure), OR department is exactly `it` or `engineering`.
  - Drops people whose position contains “blocked” titles (sales/marketing/finance/legal/product management/recruiter/hr/operations/risk/business/revenue/growth).

This is why “Number of Leads” and “Location” from the form may not change results unless you also adjust the filtering/search logic.

### Search Leads (HTTP Request) configuration checklist

Open `Search Leads` and verify:
- `Method`: `POST`
- `URL`: `https://api.hunter.io/v2/domain-search`
- Query parameters:
  - `domain`: should be `={{ $json['Domain Name'] }}`
  - `seniority`: should be `executive, senior` (or whatever you want)
  - `api_key`: set it to your Hunter API key (ideally via a secret/env var)

If you use an environment variable in n8n, a common pattern is:
- `api_key = {{$env.HUNTER_API_KEY}}`

## Groq (AI) setup

The workflow uses:
- Node: `Groq LLM (Fast AI)` with model `llama-3.3-70b-versatile` set in the export
- Node: `AI Cold Email Writer` (`@n8n/n8n-nodes-langchain.agent`)

You’ll need a Groq API key (from [console.groq.com](https://console.groq.com/)).

The prompt is highly specific to your template:
- It writes cold emails from “Adarsh Gella” to engineering leaders.
- It includes a pool of FACTS (A–F) and banned phrases.
- Output is constrained to raw JSON only:
  - `Email Subject`
  - `Email Body`

### How to customize the prompt

Open `AI Cold Email Writer` and edit the `text` field:
1. Replace the opening/intro facts with the facts you want to use.
2. Update the portfolio URL and the “would love 10 minutes to chat” line if needed.
3. If you want a different subject format or different banned phrases, update the instructions section.

Because the workflow’s `Save Email to Sheet` node expects `output['Email Subject']` and `output['Email Body']`, keep those keys exactly.

## Gmail setup

The workflow sends using:
- Node: `Send Cold Email via Gmail` (`gmail` node)
- Credential: `Gmail account` (OAuth2)

You must configure:
1. Gmail OAuth2 credential in the node.
2. `senderName` (currently `Your Name` in the export).
3. Verify `sendTo` uses the correct sheet header (`Email Address`, not `Email Address `).

## Outreach status state machine (critical)

The workflow intends this state flow:
1. During lead scrape: set `Outreach Status = Pending`
2. During AI generation: write `Email Subject` + `Email Body`
3. Before sending: set `Outreach Status = Mail Generated`
4. Email sending loop: `Filter1` selects `Outreach Status = Mail Generated`
5. After sending: update status again in `Update the Outreach Status`

In the current export, `Extract People Data` sets `Outreach Status: "Pending"`.

However (as exported):
- `Save Email to Sheet` (as exported) only writes `Email Subject` and `Email Body` (no explicit `Outreach Status = Mail Generated` mapping).
- `Filter1` selects `Outreach Status = Mail Generated`

### What to do in n8n so the send loop runs

To make the send loop work end-to-end, you want to ensure the sheet transitions like this:
`Pending` (after lead scrape)
-> `Mail Generated` (after AI generation)
-> `Sent` (after Gmail send)

Update one of these so `Outreach Status` becomes `Mail Generated` after AI email generation:
1. Option A (recommended): modify `Save Email to Sheet` to also set `Outreach Status = Mail Generated`.
   - In `Save Email to Sheet`, add a mapping for `Outreach Status`.
2. Option B: modify `Filter1` to select `Outreach Status = Pending` instead of `Mail Generated`.
3. Option C: set `Outreach Status` manually in the sheet after AI generation.

Without one of the above, the send loop will likely match zero rows.

Also make sure `Update the Outreach Status` changes the status after sending (otherwise you may resend repeatedly):
1. Open `Update the Outreach Status`.
2. Configure it to update `Outreach Status` for the current row to something like `Sent`.
3. Keep `Filter1` targeting only `Mail Generated` rows so the loop stops after each send.

## Step-by-step setup in n8n (detailed)

1. Import the workflow
   - In n8n: `Workflows` → `Import from file`
   - Select `B2B Lead Generation.json`

2. Configure credentials
   - Hunter: either replace the `api_key` query value in `Search Leads`, or move it to a secret/env var
   - Google Sheets OAuth2: `Credentials` → add `Google Sheets OAuth2` → authorize with a Google account that has access to the target spreadsheet
   - Groq credential: `Credentials` → add `Groq API` (or similarly named Groq credential) → paste your Groq API key and save
   - Gmail OAuth2: `Credentials` → add `Gmail OAuth2` → authorize the sender account you will email from

3. Configure Google Sheet tab + headers
   - Ensure the spreadsheet and tab name match the export:
     - document id `1cGZWNhzSSani12XS9wvPwMwd_4VPmRgV3TY4FFs5ZTA`
     - tab `Master Lead Generation sheet`
   - Ensure headers are exactly:
     - `ID`, `Full Name`, `Job Title`, `Company`, `Email Address`, `Phone Number`, `Profile URL`, `Company Website`, `Industry Genre`, `Outreach Status`, `Email Subject`, `Email Body`

4. Connect/fix the sheet->email matching (whitespace)
   - Open `Filter Pending Leads` and verify it references the same header names you have in Sheets (`Email Address`, `Outreach Status`).
   - Open `Send Cold Email via Gmail` and change `sendTo` to use `['Email Address']` (no trailing whitespace).
   - Open `Filter1` and change its `leftValue` to use `['Outreach Status']` (no trailing whitespace).

5. Ensure email sending has something to send
   - Decide how you want to mark “mail ready”:
     - modify `Save Email to Sheet` to set `Outreach Status = Mail Generated`, or
     - modify `Filter1` to check `Pending`, or
     - manually update the sheet.

6. Customize the email prompt
   - Open `AI Cold Email Writer` and update facts, portfolio link, and any “banned phrases”.

7. Node-by-node verification (recommended before your first real run)

Verify these nodes in order:
- `Lead Generation Form`: fields exist exactly as in the form node:
  - `Company Name`, `Location`, `Number of Leads`, `Domain Name`
- `Search Leads`: returns Hunter’s response including `data.emails[]`
- `Code in JavaScript`: outputs only “technical” candidates (filtered list)
- `Extract People Data`: outputs rows with:
  - `ID` (email address)
  - `Email Address`
  - `Outreach Status = Pending`
- `Save Leads to Sheet`:
  - `sheetName` equals `Master Lead Generation sheet`
  - `matchingColumns` uses `ID`
- `Fetch Leads from Sheet`:
  - reads from the same sheet/tab
- `Filter Pending Leads`:
  - checks `Email Address` is not empty
  - checks `Outreach Status = Pending`
- `AI Cold Email Writer`:
  - produces JSON with `output.Email Subject` and `output.Email Body`
- `Save Email to Sheet`:
  - writes `Email Subject` and `Email Body` into the correct columns
  - (if you want automatic sending) also updates `Outreach Status = Mail Generated`
- `Fetch Leads for Mail` + `Filter1`:
  - `Filter1` selects rows where `Outreach Status = Mail Generated`
- `Send Cold Email via Gmail`:
  - `sendTo` references the correct header (`Email Address`)
- `Update the Outreach Status`:
  - after sending, update `Outreach Status` to something like `Sent` (so you don’t resend)

8. Run Trigger 1 end-to-end (lead scrape)
   - Execute `Lead Generation Form` once with:
     - `Company Name`
     - `Location`
     - `Number of Leads`
     - `Domain Name`
   - Confirm your sheet gets new rows with:
     - `ID` populated (email address)
     - `Email Address`
     - `Outreach Status = Pending`

9. Run Trigger 2 end-to-end (generate and send)
   - Execute `When clicking Execute Workflow` (manual).
   - In execution logs, confirm:
     - `Filter Pending Leads` selects rows
     - `AI Cold Email Writer` produces `Email Subject` and `Email Body`
     - `Save Email to Sheet` writes them to the sheet
     - the sheet rows you expect are selected by `Filter1`
     - Gmail send executes (and `Update the Outreach Status` runs)

## Troubleshooting

### “No emails were sent”
Most common causes:
1. `Filter1` matches zero rows (status mismatch: `Pending` vs `Mail Generated`).
2. `sendTo` references a header with trailing whitespace (`Email Address ` vs `Email Address`).
3. Gmail node credential isn’t connected.

### “AI runs but the sheet isn’t updated”
1. Check `Save Email to Sheet` column mappings.
2. Confirm the AI node output parser produces `output['Email Subject']` and `output['Email Body']`.

## Notes

- The workflow export contains older Apollo-related nodes. They are not part of the Hunter-driven “Lead scrape” branch shown in the connections.
- Review the n8n execution logs at least once after your configuration changes, so you can verify the state transitions in your sheet.

## License

No license file is included in this export.

