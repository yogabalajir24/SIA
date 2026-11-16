# Open PO Monitoring & Progression - MVP

CAP (Node.js) backend + freestyle SAPUI5 app, laid out like the slide 6 prototype
(KPI tiles, PO list with status chips and AI recommendation, selected-case panel, embedded AI assistant).

## Run in SAP Business Application Studio

1. Upload `openpo-mvp.zip` to your BAS workspace and unzip it (`unzip openpo-mvp.zip`), or drag the folder in.
2. Open a terminal in the project folder:
   ```
   npm install
   cds watch
   ```
3. Open the web application link from the "Open PO Progression" entry on the cds welcome page
   (path `/openpo/webapp/index.html`). The in-memory SQLite database is created and seeded on every start.

`npm test` runs the rule, S/4 mapping, e-mail wording, service and ingest tests (60 in total).

Demo data is dated around **30 Oct 2026**. To pin "today" for the rules (7-day ASN window, reminder due date):
`OPENPO_TODAY=2026-10-30 cds watch`.

## What is in the MVP (matches the scope table)

| # | Capability | MVP status |
|---|-----------|-----------|
| 1 | Fiori-style PO progression app | Built (this app) |
| 2 | Real-time Open PO retrieval from S/4HANA via API Management | An Integration Suite iFlow reads the open POs and calls the CAP action `upsertOpenPOs` (design: `docs/IFLOW_DESIGN.md`). Today the Refresh button loads a mock of the S/4 responses through the same code |
| 3 | Dashboard, filtering, monitoring | KPI tiles, search, status filter |
| 4 | AI-generated supplier follow-up and email via Microsoft 365 | One consolidated request / reminder per supplier. Mail via Graph (`OPENPO_MAIL_MODE=graph`), default: simulated and logged |
| 5 | Direct reading of supplier replies from Microsoft 365 | Not included (as in the scope table) |
| 6 | AI extraction of Qty, ETD, ETA, ASN from the supplier response | Purchasing pastes the reply text; AI Core extracts (`OPENPO_AI_MODE=aicore`), default: built-in parser |
| 7 | Delivery / inventory impact and AI recommendation | FRS 4.4 matrix, 1-month STO stock-out rule, Critical SKU (AV, AE, BV), 7-day ASN rule, split shipments |
| 8 | PO owner review, approval and progression tracking | Accept / Reject / Override with reason code, audit trail |
| 9 | Update PO due date in S/4HANA via API | Not included. Accepted dates raise "SAP Update Required"; Purchasing amends the PO and confirms with Track SAP Update (FRS 5.2 Option 2) |
| 10 | PRD deployment, SIT/UAT, support | Not included |

## Open PO data: iFlow to PostgreSQL

* The iFlow (or, for the demo, the Refresh button) stores **PO details only**: PO number, supplier number and name, line item, material number and description, requested and delivered quantity, delivery date, delivered flag, PO creation date, PO owner. A new PO shows on the front page with nothing else.
* The supplier request text and date, the supplier response, ETA, quantity, ETD, category (outcome) and AI recommendation are written by the app, and only when the supplier replies; the validation also runs only then.
* POs that are no longer open in S/4HANA are marked as delivered. They stay in the table and leave the front page.
* Mock feed: `srv/mock/s4-open-pos.json`. Mapper: `srv/lib/s4-mapper.js`. Full design, field mapping and the iFlow script: `docs/IFLOW_DESIGN.md`.

### PostgreSQL

The model works on SQLite (local demo) and PostgreSQL. For PostgreSQL run with `CDS_ENV=postgres` and a bound PostgreSQL service on BTP. Locally:

```
export CDS_CONFIG='{"requires":{"db":{"kind":"postgres","credentials":{"host":"127.0.0.1","port":5432,"user":"...","password":"...","database":"poagent"}}}}'
NODE_ENV=production npx cds deploy
NODE_ENV=production npx cds-serve
```

## Switching on the real integrations

Create BTP destinations and set environment variables (BAS: `default-env.json` or run configuration; BTP: app environment).

| Integration | Destination | Environment variables |
|---|---|---|
| S/4HANA via API Management | `S4_APIM_OPEN_PO` | `OPENPO_S4_MODE=apim`, optional `OPENPO_S4_PATH` |
| Microsoft 365 Graph (sendMail) | `M365_GRAPH` (OAuth2 client credentials, URL `https://graph.microsoft.com`) | `OPENPO_MAIL_MODE=graph`, `OPENPO_MAILBOX=<shared mailbox address>` |
| SAP AI Core (generative AI hub) | `AI_CORE_GENAI` | `OPENPO_AI_MODE=aicore`, `OPENPO_AI_DEPLOYMENT_ID`, optional `OPENPO_AI_RESOURCE_GROUP`, `OPENPO_AI_PATH` |

The three real-mode calls were **not run** against live systems when this was built. Check the S/4 field mapping
(due date and GR quantity come from schedule-line and history data, not from the item API), the Graph permission
(`Mail.Send`), and the AI Core chat-completions path for your model.

## HANA-ready

The CDS model is database neutral. When you want HANA Cloud: `cds add hana --for production`, then deploy
(`cds deploy --to hana` or an MTA build). Add `cds add xsuaa,approuter,mta` for a full BTP deployment.

## Decisions worth confirming

* FRS 4.4.3 has a generic row "Critical SKU Delay -> Manual Review" that contradicts the row
  "Minor Delay + Sufficient Stock + Critical SKU -> Auto Accept". The code follows the specific rows.
  Set `STRICT_CRITICAL_DELAY = true` in `srv/lib/rules.js` to force Manual Review for any delayed critical SKU.
* "Insufficient stock" is Stock on Hand below Safety Stock; Projected Stock Out is Stock on Hand below open STO demand
  due within 31 days. Both only apply when the ETA is later than the PO due date.
* Override reason codes are placeholders (FRS 5.6 says the final list is still open).
* The MVP handles one PO per pasted supplier reply, no Excel attachments, no weekly/Wednesday schedulers and no
  15-minute ZM44 retry. Roles and authorization are not configured (local mock user).

## Deploy the demo to SAP BTP (Cloud Foundry)

Public link, no login, in-memory sample data. The same steps redeploy later changes.

1. Log in and pick your org and space (BAS: use the Cloud Foundry sign-in, or the terminal):
   ```
   cf login --sso -a https://api.cf.ap11.hana.ondemand.com
   cf target
   ```
2. From the project root (where `manifest.yml` is):
   ```
   cf push
   ```
   The output ends with the app route. The demo link is `https://<route>/`.
3. Optional, switch on AI Core (keeps the secret out of the code):
   ```
   cf set-env po-progression-agent OPENPO_AI_MODE aicore
   cf set-env po-progression-agent OPENPO_AI_DEPLOYMENT_ID <deployment id>
   cf set-env po-progression-agent OPENPO_AI_RESOURCE_GROUP default
   cf set-env po-progression-agent OPENPO_AI_API_URL <AI_API_URL from the service key>
   cf set-env po-progression-agent OPENPO_AI_TOKEN_URL <url from the service key>/oauth/token
   cf set-env po-progression-agent OPENPO_AI_CLIENT_ID '<clientid>'
   cf set-env po-progression-agent OPENPO_AI_CLIENT_SECRET '<clientsecret>'
   cf restage po-progression-agent
   ```
4. Later changes: edit the files, then run `cf push` again. The URL stays the same, and values set with `cf set-env` are kept.

Good to know for the demo:
* Data is in memory. Every restart or `cf push` resets it to the sample data, and everyone who opens the link sees the same data.
* Anyone with the link can use the app and the AI (it uses your AI Core quota). Do not switch on Microsoft 365 mail sending on this public copy.
* Going live later: add HANA (`cds add hana`), real destinations for S/4HANA and Microsoft 365, and a login (`cds add xsuaa,approuter`), then deploy with `cf push` or an MTA build. The link can stay the same.
# SIA-PO-AGENT
# SIA
