# Intake record guard (n8n)

An n8n workflow that sits between your AI intake agent and everything downstream. The agent
posts its JSON record here; the workflow checks it and pages a human when it has to.

I built it after my own agent logged a suspected gas leak with a null address and a null
callback number. It told the caller to get out of the house, which was right, and then handed
the on-call dispatcher an emergency nobody could act on. The full story, including the eight
other ways my kit contradicted itself:
https://dev.to/rizkynandapr/8-of-my-ai-agents-30-test-calls-failed-every-one-was-my-fault-3kff

## What it does

| Record | What happens |
|---|---|
| Emergency, no usable callback number or address | Pages a human now, says exactly what's missing |
| Emergency, complete | Pages a human with the address and callback number |
| Not an emergency, but missing or placeholder fields | Flags it for review |
| Everything | Continues to your CRM or sheet |

"Placeholder" means values that look filled in and aren't: `unknown`, `N/A`, `123 Main St`,
`555-555-5555`, `John Doe`. An agent that invents a placeholder address is worse than one that
leaves it null, because nothing downstream notices.

The webhook also replies with `severity` (`page`, `review` or `ok`) and the list of problems,
so your voice or SMS platform can log the result.

## Setup

1. In n8n: **Workflows → Import from file**, pick `intake-record-guard.json`.
2. Open **Alert on-call** and paste a Slack incoming webhook URL. For Discord, rename `text`
   to `content` in the JSON body. Anything that accepts a POST works.
3. Replace **Send to your CRM or sheet** with your real destination. Both branches reach it,
   so flagged records are stored too, not just alerted.
4. Activate the workflow.

No credentials are included. Nothing in the file needs a key until you add your own nodes.

## Test it

PowerShell:

```powershell
$body = @{
  schema_version = "1.0"; trade = "hvac"; urgency = "emergency"; emergency_type = "gas"
  callback_number = $null
  service_address = @{ street = $null; city = $null; postal_code = $null }
  problem_summary = "Caller said the furnace isn't working and it smells like eggs downstairs."
  outcome = "escalated"
} | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "YOUR_WEBHOOK_URL" -ContentType "application/json" -Body $body
```

bash:

```bash
curl -X POST "YOUR_WEBHOOK_URL" -H "Content-Type: application/json" -d '{
  "schema_version": "1.0", "trade": "hvac", "urgency": "emergency", "emergency_type": "gas",
  "callback_number": null,
  "service_address": {"street": null, "city": null, "postal_code": null},
  "problem_summary": "Caller said the furnace is not working and it smells like eggs downstairs.",
  "outcome": "escalated"
}'
```

Expected reply:

```json
{ "severity": "page", "problems": ["no usable callback_number", "no usable service_address"] }
```

## Field names

It expects `urgency` (`emergency`, `same_day`, `routine`), `callback_number`,
`service_address` (an object) and `problem_summary`. If your agent uses different names,
change them in the **Check record** node. The record can be the whole request
body or wrapped as `{ "record": { ... } }`.

## License

MIT. Use it in client work, change it, ship it.
