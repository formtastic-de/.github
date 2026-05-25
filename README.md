# Formtastic

**[app.formtastic.de](https://app.formtastic.de)** — Digital form management platform for creating, filling, and managing forms.

---

### Batch Export (CSV)

Export form submissions in bulk as a CSV file. Authentication is passed as a query parameter.

```
GET https://app.formtastic.de/export-bulk-external.csv?token=<token>&template=<id>&state=<state>&from=<YYYY-MM-DD>&to=<YYYY-MM-DD>
```

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `token` | string | yes | Your API authentication token |
| `state` | string | yes | Form data state: `sent_and_saved`, `filled`, `draft`, or `rejected` |
| `template` | integer | yes | Form template ID |
| `from` | string | yes | Start date (`YYYY-MM-DD`) |
| `to` | string | yes | End date (`YYYY-MM-DD`) |
| `transposed` | boolean | no | Use transposed layout (default `false`) |
| `onlyOnlineForms` | boolean | no | Only include online form submissions (default `false`) |

**Response** — `200 OK` — CSV file download.

**Errors**

| Code | Meaning |
|---|---|
| 400 | Missing or invalid parameters |
| 401 | Invalid token |
| 403 | Export feature not available in your plan |

---

### Webhooks

Webhooks are how Formtastic pushes form-submission events to your own systems in near real time.
Whenever a submission changes state (for example, when it is saved or assigned), Formtastic sends
an HTTP `POST` request to a URL you control.

#### At a glance

- **Transport**: HTTP `POST` with `Content-Type: application/json`.
- **Trigger**: a state change on a form submission of a template you have a webhook configured for.
- **Body**: a JSON object describing the form, the submission and the user who triggered the change.
- **Authentication**: each webhook has its own secret token, sent with every request.
- **Retries**: failed deliveries are retried automatically; your endpoint must be idempotent.

#### Payload Schema

A delivery is a single `POST` to your URL with a JSON body of the following shape:

```json
{
  "form": {
    "id": 10631,
    "name": "All fields"
  },
  "run_id": "9543c085-0c19-431f-b654-0b0e970063d1",
  "form_data": {
    "id": 435666,
    "data": {
      "input_text_with_default": "Default",
      "input_integer_1_to_5": 1,
      "check": false,
      "...": "field values keyed by field placeholder name"
    },
    "state": "sent_saved",
    "create_time": "2026-04-27T13:50:49.127075+00:00",
    "update_time": "2026-04-27T13:50:49.382868+00:00"
  },
  "timestamp": "2026-04-27T13:50:49.400034+00:00",
  "triggered_by": {
    "id": 83,
    "email": "user@example.com"
  },
  "lifecycle_state": "sent_saved"
}
```

#### Field Reference

| Field | Type | Description |
|---|---|---|
| `form.id` | number | The form template ID. |
| `form.name` | string | The template name at trigger time. |
| `run_id` | string | A UUID identifying this delivery. Use it for **idempotency**. |
| `form_data.id` | number | The submission ID. |
| `form_data.data` | object | The submission's field values, keyed by field placeholder name. The shape depends on the form. |
| `form_data.state` | string | The submission's lifecycle state at trigger time. |
| `form_data.create_time`, `form_data.update_time` | string | ISO-8601 timestamps for the submission. |
| `timestamp` | string | ISO-8601 timestamp when the webhook was triggered. |
| `triggered_by` | object | The user who caused the state change (id, email). |
| `lifecycle_state` | string | Same as `form_data.state`; provided at the top level for convenience. |

#### Lifecycle States

| State | Meaning |
|---|---|
| `inbox` | Newly received, not yet picked up. |
| `in_work` | Being edited by an assignee. |
| `sent_assigned` | Sent to a specific assignee. |
| `sent_saved` | Sent and saved. |
| `removed` | Deleted/archived. |

#### Authentication

Each webhook has its own **secret**, generated when the webhook is created. The same secret is included with every delivery so your receiver can verify the request's authenticity.

Your receiver should:

1. Read the secret from the incoming request.
2. Compare it (in constant time) to the secret you stored when configuring the webhook.
3. Reject the request with `401` if the values do not match.

If the secret leaks, regenerate it from the webhook detail page. The previous secret stops working immediately.

#### Responding to a Delivery

- **2xx** — the delivery is considered **successful**. No more attempts are made.
- **Anything else, or a network error** — the delivery is considered **failed** and Formtastic will retry.

Respond as quickly as you can. Long-running work should be done **after** acknowledging the request (e.g. by enqueuing a background job).

#### Retries and Idempotency

When a delivery fails, Formtastic schedules another attempt. Each attempt for the same event shares the same `run_id`, so your receiver may see the same payload more than once.

- **Deduplicate by `run_id`.** Persist the `run_id` of each successfully processed delivery and skip duplicates.
- **Make your processing idempotent.** Even without explicit deduplication, your downstream effects should tolerate being applied more than once.

A run continues retrying until one attempt succeeds, the retry budget is exhausted (the run ends in `failed` state), or an administrator cancels it.

#### Best Practices

- **Use HTTPS** with a valid certificate.
- **Verify the secret** on every request.
- **Acknowledge fast and process in the background** to keep deliveries within Formtastic's timeout window.
- **Be idempotent.** Retries are normal, not exceptional.
- **Log the `run_id` and `timestamp`** of every delivery you accept.
- **Rotate the secret** if it might have leaked, and immediately update your receiver's configuration.

---

## Links

- **Application**: [app.formtastic.de](https://app.formtastic.de)
