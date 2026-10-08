# Linear BuildShip Trigger Node
For BuildShip workflow authors who want Linear events to start downstream automation. The existing trigger definition registers a Linear webhook for selected resource types and adds a `statusChanged` flag to qualifying issue updates. Its source contract is documented below; importing and running this definition in BuildShip has not been verified.

## Get the trigger definition

```bash
git clone https://github.com/sgardoll/Linear-BuildShip-Trigger-Node.git
cd Linear-BuildShip-Trigger-Node
```

Open [linear-trigger-buildship.json](linear-trigger-buildship.json). It contains one webhook-trigger definition with its configuration, output schema and lifecycle script, not a complete BuildShip workflow or a published workflow remix link.

### Import route

The proposed trigger-specific route is **Add Trigger > New Trigger > Paste**, but importing this definition through that route has not been verified. BuildShip's [copy/paste documentation](https://docs.buildship.com/copy-paste) covers ordinary nodes and whole workflows; it does not establish this trigger-specific sequence.

## Inputs and credentials

The names, options and defaults below come from the definition's `config`, `_groupInfo` and embedded script. Select the native **API Key** credential; the script reads it through BuildShip's `auth.getKey()`, not an `apiKey` text input. Use a Linear personal API key whose owner is a workspace admin. [Linear requires admin access to create/read webhooks](https://linear.app/developers/webhooks). Do not put a key or signing secret into the JSON, examples or logs.

| Input label | Source identifier | Selection/default and effect |
| -- | -- | -- |
| API Key | `auth.getKey()`; `apiKey` in `config.required` | Required native credential selection. Missing/empty selection throws an error. |
| Action | `action` | Default **All changes** (`all`). Other options: **Created** (`create`), **Updated** (`update`), **Removed** (`remove`), **Issue status changed** (`statusChanged`). The last option requires `Issue` in Resource Types. |
| Resource Types | `resourceTypes` | Default `Issue`. Supported selections: `Issue`, `Comment`, `IssueLabel`, `Project`, `ProjectUpdate`, `Cycle`, `Reaction`. At least one is required; registration subscribes to these types. |
| Team ID | `linearTeamId` | Default empty: all public teams. For a specific team, use its UUID. The schema also describes a team key, but the script passes the string directly to Linear without resolving it; key acceptance is unverified. Existing webhook updates do not change team scope. |
| Webhook Label | `webhookLabel` | Default `BuildShip Workflow`; blank/whitespace also falls back to this label. |
| Resolve State Names | `resolveStateNames` | Default `true`. On qualifying status updates, optionally looks up state names with a 2-second request timeout. Set `false` to skip that lookup; a state object already in the event is still retained. |

The script constructs the execution URL and requires public HTTPS without URL credentials, query or fragment. It stores the Linear-generated webhook ID and signing secret under `linearWebhookId` and `linearWebhookSecret`.

## Event and output contract

The embedded `onExecution` function verifies the original request bytes against the `Linear-Signature` HMAC-SHA256 signature and checks `webhookTimestamp` before filtering. Linear's [webhook documentation](https://linear.app/developers/webhooks) describes these payload and verification fields.

The returned **Linear Event** preserves the original payload and adds:

| Field | Current script behavior |
| -- | -- |
| `statusChanged` | `true` only for an `Issue` `update` with an `updatedFrom` object containing its own `stateId` property. The script tests property presence, not inequality of the old/new IDs. |
| `previousState` | Resolved previous state object when lookup succeeds; otherwise `null`. |
| `currentState` | The event's `data.state` object if supplied, otherwise a resolved current state when available, otherwise `null`. |
| `statusTransition` | `Previous name -> Current name` only for a qualifying update with both names available; otherwise `null`, including some status changes. |

**Action** filters execution: `all` accepts every verified event; `create`, `update` and `remove` match the payload action; `statusChanged` matches the predicate above. An unmatched event calls BuildShip's `terminate(200, ...)` and does not return a downstream event. Resource Types filters the Linear subscription; the execution function does not separately recheck resource type membership.

### Redacted example

For **Action = Issue status changed**, **Resource Types = Issue**, **Resolve State Names = false**, consider this illustrative payload:

```json
{
  "action": "update",
  "type": "Issue",
  "data": {
    "id": "[redacted-issue-id]",
    "stateId": "[redacted-current-state-id]",
    "state": {
      "id": "[redacted-current-state-id]",
      "name": "In Progress"
    }
  },
  "updatedFrom": { "stateId": "[redacted-previous-state-id]" },
  "url": "[redacted-issue-url]",
  "webhookTimestamp": 1791360000000
}
```

After successful verification, the source would preserve those fields and append the following derived fields. This is a source-derived example, not a recorded delivery or a runnable request: identifiers are redacted, the timestamp is fixed and the signature is omitted. A real delivery needs a matching signature and a timestamp within 60 seconds of the receiving clock.

```json
{
  "statusChanged": true,
  "previousState": null,
  "currentState": {
    "id": "[redacted-current-state-id]",
    "name": "In Progress"
  },
  "statusTransition": null
}
```

The separate `onResponse` function specifies HTTP 200 with `{ "received": true }` and no caching. That acknowledgment is distinct from the event returned to downstream nodes and does not demonstrate successful downstream automation.

## Lifecycle, failures and limits

- If a stored webhook ID has no stored secret, registration attempts to recover its signing secret from Linear. If creation returns an ID without a secret, it preserves the ID and throws to avoid creating a duplicate on the next attempt. Missing credential/provider context, invalid configuration or a non-public URL also prevents registration.
- Execution rejects a missing stored signing secret or unavailable raw request body with status 500. A missing/malformed/mismatched signature or missing/invalid timestamp, including one more than 60 seconds in either direction from the receiving clock, is rejected with 401. Invalid JSON after verification is rejected with 400. Filtering requires BuildShip's `terminate` helper for unmatched events.
- Optional state-name lookup errors are logged without discarding the verified event. States and transitions can remain `null`; downstream nodes must handle that. Ordinary registration requests have a 10-second timeout. There is no event deduplication or delivery queue in this script.
- The script logs the full derived event. `getData` reads recent trigger output from Google Cloud Logging for preview; it depends on the provider's logging context and available entries and can fail or show a previous event. It does not request a new Linear delivery. Workspace payloads may contain sensitive content; this example's redaction does not redact runtime event logs.

This repository supplies a trigger definition, not downstream workflow logic or a coordinator. Source inspection and JSON/Markdown checks do not establish import compatibility, live webhook delivery or coordinator operation.
