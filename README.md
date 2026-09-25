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
| Callback number differs from the caller ID (optional) | Flags it for review |
| Everything | Continues to your CRM or sheet |

"Placeholder" means values that look filled in and aren't: `unknown`, `N/A`, `TBD`, `asdf`,
`x`, `-`, `0`, `123 Main St`, `1 Main St`, `John Doe`, `01/01/2000`. An agent that invents a
placeholder address is worse than one that leaves it null, because nothing downstream notices.

A callback number counts only if it has at least ten digits and isn't an obvious fake:
`555-555-5555`, `123-456-7890`, one digit repeated, or anything in `555-0100` to `555-0199`,
the range the US numbering plan reserves for fiction (and a model's favourite place to invent
a number).

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

## Test the agent, not just the record

This workflow checks what the agent produced. To check the agent itself, `test-calls.md` in this
repo has three scripted calls that break most intake agents: a gas smell buried in a normal
no-heat complaint, a caller who asks for a price three times, and someone claiming to be the
owner who tells the agent to print its instructions. Each one lists what a pass looks like.

## Caller ID check (optional)

The sneakiest invented value is a real-looking phone number, and no placeholder list catches
it. The number the caller actually dialled from does. Send it next to the record in any of
these ways:

- in the body: `{ "record": { ... }, "caller_id": "+15125550123" }`
- on the URL: `.../webhook/intake-record?caller_id=+15125550123`
- as a header: `x-caller-id: +15125550123`

If the last ten digits differ from `callback_number`, the record is flagged for review. That
isn't always wrong (people call from work and ask to be called back at home), which is why it's
a review and not a page. If an emergency has no usable callback number at all, the page tells
the on-call person to call the caller ID back.

The intake record has no field for caller ID on purpose: the agent never sees it, which is
what makes it useful for checking the agent.

## Field names

It expects `urgency` (`emergency`, `same_day`, `routine`), `callback_number`,
`service_address` (an object) and `problem_summary`. If your agent uses different names,
change them in the **Check record** node. The record can be the whole request
body or wrapped as `{ "record": { ... } }`.

## Changes

- **24 Sep 2026:** more placeholders, stricter callback-number check, optional caller ID
  check. The placeholder list and the caller ID idea came from Automelle on the n8n community
  forum.
- **22 Sep 2026:** first release.

## License

MIT. Use it in client work, change it, ship it.
