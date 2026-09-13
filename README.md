# n8n Thailand Visa Monitor

Production-style n8n workflow that monitors selected official Thai government sources for visa-exemption changes affecting holders of ordinary Russian passports.

The project fetches official pages directly, converts the HTML into compact text, asks OpenAI to classify the current and future visa rules, compares the result with the previous state, and sends Telegram alerts only when the factual state changes. A daily heartbeat confirms that the monitor is still running.

## What it demonstrates

- scheduled automation in n8n;
- direct HTTP retrieval from official sources;
- HTML cleanup and context preparation;
- structured LLM classification with the OpenAI Responses API;
- state persistence with n8n Data Tables;
- fingerprint-based deduplication;
- conditional Telegram notifications;
- daily health/status messages;
- a separate production error workflow.

## Architecture

```text
Schedule Trigger (06:00 / 12:00 / 18:00 Europe/Moscow)
        |
        +--> Fetch Thai Embassy Moscow ----+
        |                                  |
        +--> Fetch Thai Consular MFA ------+
                                           |
                                Wait For Official Sources
                                           |
                                Prepare Official Context
                                           |
                                  Classify Visa Status
                                  (OpenAI Responses API)
                                           |
                                Parse Visa Classification
                                           |
                                  Get Previous State
                                           |
                              Compare With Previous State
                                  |                  |
                                  |                  +--> Update State
                                  |
                                  +--> Route Notification
                                         | changed  -> Telegram urgent alert
                                         | unclear  -> Telegram warning
                                         | no_change
                                               |
                                          18:00 only
                                               |
                                          Daily status
```

Production execution errors are handled by a separate workflow:

```text
Error Trigger -> Prepare Error Message -> Telegram Error Alert
```

## Key design decisions

### Official sources first

The workflow downloads the source pages directly instead of asking the model to discover and judge the whole web in one long request. This makes classification faster, easier to debug, and less dependent on search behavior.

Current source pages:

- Royal Thai Embassy in Moscow
- Thailand Department of Consular Affairs

### Structured classification

The model must return a strict JSON schema including:

- `status`: `no_change`, `changed`, or `unclear`;
- current stay rule;
- future rule, if confirmed;
- effective date;
- decision status;
- confidence.

The model is instructed to use only the official text supplied by the workflow.

### Fingerprint deduplication

Free-form summaries are not used to decide whether the state changed.

Instead, the workflow creates a stable fingerprint from factual fields:

```text
status | current_stay_days | future_stay_days | effective_date | decision_status
```

This prevents repeated alerts when the model rewrites the same facts in different words.

### Daily heartbeat

No-change executions stay quiet during the morning and daytime checks. At 18:00 the workflow sends a short daily status so the operator can see that the monitor is alive and the sources were checked.

## Repository structure

```text
workflows/
  01-thailand-visa-monitor.json
  02-error-handler.json

docs/
  architecture.md
  setup.md

sample-data/
  visa_monitor_state.csv
```

The workflow exports are sanitized. They do not contain API keys, Telegram credentials, private chat IDs, n8n instance IDs, or the author's Data Table ID.

## Setup

See [`docs/setup.md`](docs/setup.md).

## Scope and limitation

This version monitors the specific official pages configured in the HTTP Request nodes. It is intentionally not a general-purpose web crawler. If Thai authorities publish future changes only on new URLs without updating the monitored pages, the source list should be updated or supplemented with a separate discovery workflow.

## Stack

- n8n
- OpenAI Responses API
- GPT-5.6 Sol
- n8n Data Tables
- Telegram Bot API
- JavaScript / Code nodes
- HTTP Request nodes

## Portfolio use

This is a demonstration of a reliable monitoring pattern that can also be adapted for:

- regulations and legislation;
- price or tariff changes;
- vendor status pages;
- product documentation;
- government announcements;
- compliance monitoring;
- internal business rules.

## Security

Keep all secrets in n8n Credentials. Do not hard-code OpenAI API keys or Telegram bot tokens inside workflow JSON exports.
