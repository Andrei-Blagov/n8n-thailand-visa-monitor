# Architecture

## Main workflow

### 1. Schedule

`Schedule Trigger` runs at 06:00, 12:00 and 18:00 in the configured n8n timezone.

### 2. Official source retrieval

Two HTTP Request nodes fetch official pages in parallel:

- `Fetch Thai Embassy Moscow`
- `Fetch Thai Consular MFA`

Both return the page as text.

### 3. Context preparation

`Wait For Official Sources` waits for both branches.

`Prepare Official Context`:

- removes scripts, styles, SVG, comments and most HTML markup;
- normalizes whitespace;
- limits document size;
- combines both sources with explicit URL labels.

### 4. LLM classification

`Classify Visa Status` sends the official context to the OpenAI Responses API.

The model is not asked to search the internet. Its only job is to extract and classify facts from the supplied official documents.

The response uses a strict JSON schema with:

- `has_change`;
- `status`;
- `current_rule`;
- `current_stay_days`;
- `future_rule`;
- `future_stay_days`;
- `effective_date`;
- `decision_status`;
- `summary`;
- `confidence`.

### 5. Normalization and guardrails

`Parse Visa Classification`:

- parses the structured response;
- attaches the official source URLs;
- rejects an unconfirmed `changed` state;
- builds a factual state fingerprint.

### 6. Persistent state

`Get Previous State` reads the last result from an n8n Data Table.

`Compare With Previous State` compares the new fingerprint with the stored fingerprint.

`Update State` stores the latest result on every successful run.

### 7. Notifications

`Route Notification` routes by `notify_type`:

- `changed` -> urgent Telegram alert;
- `unclear` -> Telegram warning;
- `no_change` -> daily-status check.

`Send Daily Status?` allows the no-change heartbeat only at 18:00.

## Error workflow

The main workflow should be configured to use `Thailand Visa Monitor Error Handler` as its Error Workflow.

On a production execution failure:

1. `Error Trigger` receives execution metadata.
2. `Prepare Error Message` formats workflow name, failed node, execution ID, error text and execution URL.
3. `Send Error Alert` sends the diagnostic message to Telegram.

## Why the workflow uses a fingerprint

Model summaries can vary from run to run even when the underlying facts do not.

Comparing summary text would therefore create false alerts.

The fingerprint contains only state fields:

```text
status|current_stay_days|future_stay_days|effective_date|decision_status
```

This makes notifications deterministic at the workflow level.
