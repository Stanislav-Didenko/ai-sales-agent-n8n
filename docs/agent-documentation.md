# Agent Documentation: Veloxa AI Sales Agent

Agent documentation (services, credentials, launch, runbook) and the go-live checklist status: what is done, what is done differently, and which risks are accepted. Status as of 18.11.2026.
n8n workflows: `workflow/veloxa_sales_agent_demo.json` (the agent) and `workflow/veloxa_evaluations.json` (the evaluation run).

## 1. Name and purpose

**Agent:** Veloxa AI Sales Agent (lead qualification). Concept project: the agency is fictional and all leads are synthetic.

**What it does:** takes inbound requests from the website form, extracts the facts with verbatim quotes, scores the lead in code (HIGH / MEDIUM / ATTENTION / LOW / STOPPED) against the service catalog, creates the deal in Pipedrive, drafts the reply or the clarifying questions plus a call brief, and puts everything in front of the sales manager in Telegram. A human sends every email.

## 2. Workflow scheme

**Main flow (web form):**
webhook → read settings / services / out-of-scope / leads → intake checks (required fields, honeypot, duplicate within 24 h, injection regex) → Claude Haiku: extract facts with quotes → Verify & score (code: quotes checked, currencies and deadlines converted, tier decided) → Pipedrive: person, org, deal, note → website enrichment (HIGH and MEDIUM only: HTTP, Browserless fallback, Haiku summary) → Claude Sonnet: draft + call brief (HIGH and MEDIUM) → guard check of the draft → Telegram card with buttons → save lead row → cost log, error alert, error log.

**Separate branches on the same canvas:**
- **Lead replied (Gmail trigger):** match the reply to the lead by thread → re-extract and re-score → new draft and card.
- **Manager actions (Telegram trigger):** Send / Edit / Later / Not a lead / Treat as High / Ask questions / Low / Nudge / Close / AI call payload, plus the `/forget email` command (right to erasure, with confirmation). Only the allowed chat ID is accepted.
- **Daily digest (schedule, 09:00):** volume by tier, AI cost, postponed cards, leads waiting for an answer, timeout cards with Nudge / Close after 3 business days.
- **Test mode:** a request with `test_run` set skips duplicates, CRM, Telegram and drafts and writes the result to the Tests sheet. Used by the Evaluations workflow.
- **Crash handler:** Error Trigger in the same workflow → Telegram "Sales agent crashed".
- **Setup (run once):** creates the Pipedrive pipeline, stages and custom fields and writes their IDs to settings.

**Evaluations workflow** (separate): "Veloxa AI — Evaluations (sales agent)" (`workflow/veloxa_evaluations.json`). It sends each case from the `eval` sheet through the live webhook in test mode and compares tier, service, budget and deadline with the expected values.

## 3. Services

| Service | Used for |
|---|---|
| n8n Cloud | Runs the workflow, webhook for the form |
| Anthropic API | Claude Haiku: fact extraction, website summary. Claude Sonnet: email draft and call brief |
| Google Sheets | Agent memory and config: settings, services, out_of_scope, leads, log (cost), errors, Tests, eval |
| Pipedrive | CRM: person, organization, deal, stages, notes, call activity |
| Gmail | Sends the emails the manager approved, receives the lead's replies |
| Telegram Bot | Manager cards and buttons, alerts, daily digest, `/forget` |
| Browserless | Renders JavaScript websites when a plain HTTP request gets no text |
| Web form (static HTML) | Input: posts JSON to the n8n webhook |

## 4. Credentials (names only, no values)

- Anthropic: `Anthropic account` (anthropicApi, used from HTTP Request nodes)
- Google Sheets: `Google Sheets OAuth2 API` (googleSheetsOAuth2Api)
- Gmail: `Gmail account` (gmailOAuth2)
- Pipedrive: `Pipedrive account` (pipedriveApi, API token)
- Telegram: `Telegram Demo Bot` (telegramApi, bot token)
- Browserless: `Browserless` (httpQueryAuth, token as a query parameter)

IDs that are not secrets but are environment-specific (sheet ID, Telegram chat ID, Pipedrive stage and field IDs) live in the `settings` sheet, not in the nodes.

## 5. Launch instructions

1. **Credentials:** connect the six credentials above. Each node references them by name; no key is typed into a node.
2. **Config:** in the `settings` sheet set `telegram_chat_id`, `agency_name`, `sender_name`, `timezone`, models and prices. Run the "Setup (run once)" branch: it creates the Pipedrive pipeline, stages and fields and writes their IDs back to settings.
3. **Publish** the workflow. Production runs use the published version only, so an edit in the draft does not work until it is published.
4. **Form:** the form posts to the **production** webhook URL `https://<n8n-host>/webhook/veloxa-lead`, not the `webhook-test` URL.
5. **Switches off:** `enrich_force_render` = no. No `test_run` field in the live form.
6. **Smoke test:** send one HIGH-type request from a test address (plus-address of a mailbox that is NOT connected to the agent). Check: a row in `leads`, a deal on the right Pipedrive stage, a Telegram card with a draft, a row in `log`, nothing new in `errors`. Press Send, then check the email arrived and the card shows "Sent".
7. **Regression:** run the Evaluations workflow and compare with the baseline (section 7).

## 6. Runbook

| Symptom | Likely cause | Action |
|---|---|---|
| No new leads, no errors | Form posts to the test webhook, or the workflow is not published | Check the form URL is `/webhook/`, not `/webhook-test/`; publish the workflow |
| Telegram buttons do nothing | Workflow not published, or the click came from another chat | Publish; check `telegram_chat_id` in settings matches the manager chat |
| Alert "Claude extract: 429 / overloaded / timeout" | Anthropic rate limit or outage | Retry (2 tries) is already on; the lead still gets a card. If frequent, check the Anthropic usage tier and balance |
| Card says "model output could not be read (needs_review)" | The model returned broken JSON | Handle the lead manually from the card (Treat as High / Ask questions). If it repeats, check the prompt and model in settings |
| Lead is ATTENTION with `possible_injection` | Regex filter or the model found instruction-like text | Expected behaviour. Read the request; route it with the buttons if it is a real client |
| Card says "Email NOT sent" | Gmail send failed (token, quota, address) | Lead stays unsent. Check the Gmail credential, then press Send again (no automatic retry on purpose: no duplicate emails) |
| Alert "PD: create …" failed | Pipedrive token expired, trial ended, rate limit | Re-authorise the credential; create the missing deal by hand from the `leads` row (no automatic retry on creates on purpose: no duplicates) |
| Alert from Google Sheets (429 / quota) | Sheets API limit (≈60 requests / min per user) | Writes retry 3×. If frequent, batch the load or move the memory to a database |
| Google credential stops working | OAuth token revoked or expired | Re-authorise in n8n Credentials |
| `site_unreachable` on many leads | Browserless token or units exhausted, or sites block bots | Check the Browserless account. The lead is processed without the summary anyway |
| Reply from the lead is not picked up | Reply sent from the agent's own mailbox, or a new thread instead of Reply | Test with an outside mailbox; the lead must reply in the same thread |
| "Sales agent crashed" alert | A node failed outside the continue-on-error path | Open the execution by ID (failed executions are saved), fix, publish, re-send the lead if needed |
| A person asks to delete their data | GDPR erasure request | In the manager chat: `/forget their@email.com` → confirm. Sheet rows anonymised, Pipedrive deal and person deleted (kept 30 days in Pipedrive trash), cards wiped. Gmail correspondence is kept as a business record |

## 7. Go-Live checklist status

✅ done · ⚠️ partial / done differently · ❌ open (accepted risk)

| # | Item | Status | How / why |
|---|---|---|---|
| 1 | Failure handler | ✅ | Error Trigger inside the same workflow + per-node continue-on-error with alerts |
| 2 | Retry only where safe | ✅ | Retry on reads, model calls, upserts by lead_id, PUT, DELETE. No retry on Pipedrive creates and Gmail sends (fixed 18.11) |
| 3 | Error branching | ✅ | Continue on error + checks in code (email marked sent only if Gmail returned an ID) |
| 4 | Fallback, nothing lost | ✅ | Unreadable output → ATTENTION + `needs_review`; any failed step → lead continues, error recorded |
| 5 | Owner notified | ✅ | Telegram alert with failed node and execution ID, current run only |
| 6 | Executions saved | ⚠️ | Failed executions saved; successful ones not stored since 18.11 (they held personal data). Soft errors are investigated from the `errors` sheet |
| 7 | Separate log | ✅ | Sheets `log` (cost per call) and `errors` |
| 8 | No sensitive data in logs | ✅ | Logs and alerts carry lead_id only; execution history no longer kept for successful runs |
| 9 | Minimal log fields | ✅ | ts, lead_id, step, model, tokens, cost, tier / where, message, execution ID |
| 10 | Evaluation dataset | ✅ | 27 cases: 14 budget / deadline formats, all 7 services, spam, out of scope, 2 disguised injections |
| 11 | Baseline recorded | ✅ | 16.11: 25/27. c16 was an error in the expected value (fixed), c14 model nondeterminism (fixed in code) |
| 12 | Regression routine | ✅ | prompt_version in every row; re-run Evaluations after any prompt / model / code change |
| 13 | Cost per lead | ✅ | From logged tokens: ≈ $0.0003 without a draft, ≈ $0.010 for HIGH (+ $0.0005 website), ≈ $0.015 for a MEDIUM → HIGH dialogue |
| 14 | Monthly cost | ✅ | At 300 leads / month with half getting a draft: AI ≈ $2 / month. Fixed costs dominate: n8n plan, Pipedrive seat, Browserless |
| 15 | Most expensive node | ✅ | Claude: draft email (Sonnet), roughly 95% of AI cost |
| 16 | Provider spend limit | ❌ | Not set yet. Action: set a monthly limit in the Anthropic console |
| 17 | Cheaper model tested | ❌ | Haiku already does extraction. Moving drafts to Haiku was not evaluated |
| 18 | Keys in Credentials only | ✅ | All six services; GitHub export and its full history checked 18.11 |
| 19 | Least privilege | ⚠️ | Sending only via the manager's button from the allowed chat; the agent never writes `manager_action`. Accepted risk: the Gmail credential can technically send; Google and Pipedrive credentials are not scoped down |
| 20 | Injection rule | ✅ | In all three prompts + regex filter before the model; tested by c25 and c26 |
| 21 | Human-in-the-loop | ✅ | Every email, reminder, stage change and call payload is a manager click |
| 22 | Privacy | ⚠️ | Notice on the form, separate call consent, `/forget` erasure (18.11). Open: consent checkbox + privacy policy link, retention period |
| 23 | Workflow scheme | ✅ | `docs/agent-flow.png`, README, section 2 |
| 24 | Services and credentials | ✅ | Sections 3–4 |
| 25 | Launch instructions | ✅ | Section 5 |
| 26 | Runbook | ✅ | Section 6 |
| 27 | Handover | ✅ | This document + README + settings sheet with a description per key |

**Accepted risks** (training agent, would be closed for a real client first): no provider spend limit (16), cheaper draft model not evaluated (17), broad credential scopes (19), no consent checkbox or retention period (22).
