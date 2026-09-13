# Setup

## 1. Import workflows

Import `workflows/02-error-handler.json` and then `workflows/01-thailand-visa-monitor.json`. The public exports are inactive by default and contain placeholder IDs.

## 2. Create the state Data Table

Create an n8n Data Table named `visa_monitor_state` with string columns: `key`, `last_status`, `last_rule`, `last_effective_date`, `last_summary`, `last_sources`, `last_checked_at`, `last_state_fingerprint`.

Add one initial row. The workflow expects numeric row ID `1`; set `key` to `thailand_russia_visa` and leave the other fields empty.

Then select the created Data Table in `Get Previous State` and `Update State`.

## 3. Configure OpenAI authentication

Create an n8n Header Auth credential for the OpenAI API and select it in `Classify Visa Status`. Restrict the credential to `api.openai.com` if your n8n setup supports domain restrictions.

The workflow uses `https://api.openai.com/v1/responses`. Keep the API key in n8n Credentials; do not place it directly in the workflow JSON.

## 4. Configure Telegram

Create or select your Telegram Bot credential. Replace `YOUR_TELEGRAM_CHAT_ID` and assign the Telegram credential in `Send Changed Alert`, `Send Unclear Alert`, `Send Daily Status`, and `Send Error Alert` in the error workflow.

## 5. Configure the Error Workflow

Open the main workflow settings and select `Thailand Visa Monitor Error Handler` as the Error Workflow. Test this path with a scheduled/production execution.

## 6. Check source URLs

The public workflow currently uses:

- `https://moscow.thaiembassy.org/en/publicservice/revision-of-thailand-s-visa-exemption-and-visa-on-arrival-schemes`
- `https://consular.mfa.go.th/th/content/1-9-69-00`

If the official authorities move or replace these pages, update the HTTP Request nodes.

## 7. Schedule

Default cron expression: `0 0 6,12,18 * * *`.

The daily heartbeat checks `Europe/Moscow` time and is sent at 18:00.

## 8. Test before publishing

Verify that both official pages return HTML, context cleanup works, the classifier returns structured JSON, the Data Table updates, repeated identical states do not repeat urgent alerts, the 18:00 heartbeat is sent, and a deliberate production error triggers the error workflow.

## 9. Publish

After all credentials, IDs, and the Error Workflow are configured, publish/activate both workflows.
