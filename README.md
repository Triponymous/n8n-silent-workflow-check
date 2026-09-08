# Silent workflow check for n8n

Finds n8n workflows that are **active but have not run**.

That failure produces no failed execution, so an error workflow never fires for it. It happens
after a restart that did not re-register the schedule trigger, or when someone toggles a workflow
off while debugging and forgets it. The execution list looks clean and nothing has run for days.

## What it does

Reads your own instance through the n8n API, takes every active workflow, finds its last
execution, and returns the ones that have been quiet longer than you allow.

Five nodes: schedule trigger → config → two HTTP requests → one code node. Attach your own alert
node at the end.

```
Every morning → Configuration → Get workflows → Get executions → Find silent workflows
```

Output per stalled workflow:

```json
{
  "workflow": "Nightly invoice sync",
  "workflowId": "aBcD1234",
  "lastRun": "2026-09-05T02:00:11.000Z",
  "hoursSilent": 74,
  "note": "active but stalled"
}
```

When nothing is stalled it returns a single `{ "silent": 0, "activeChecked": 12 }`, so you can
branch on that before alerting.

## Setup

1. In n8n: **Settings → n8n API → Create an API key**.
2. Import `silent-workflow-check.json` via **Workflows → Import from File**.
3. On **Get workflows**, create a *Header Auth* credential:
   - Name: `X-N8N-API-KEY`
   - Value: your API key

   Select the same credential on **Get executions**.
4. In **Configuration**, set `baseUrl` to your n8n address without a trailing slash, and
   `staleHours` to how long silence is still normal for you.
5. Attach your alert node after **Find silent workflows** and activate the workflow.

## Known limits

Both of these are real, and the first one is the reason external monitoring exists at all.

- **It lives inside the instance it watches.** If that instance is down, mid-restart, or its
  schedule triggers never re-registered, this check does not run either — and its silence looks
  exactly like everything being fine. A self-check cannot cover the case where the checker is
  the thing that failed.
- **It reads the last 250 executions.** On a busy instance, a workflow that went quiet weeks ago
  can fall out of that window. Raise the limit in both HTTP nodes, or run the check more often.

## Six more failures that produce no error

Stalled schedules are one of seven. The others: downstream rate limits, credentials that expired,
webhooks nobody calls any more, workflows switched off during debugging, execution history pruned
by `EXECUTIONS_DATA_MAX_AGE`, and daylight saving shifting a schedule by an hour twice a year.

Written up with the one check that catches each, at
[duskwatch.me/checklist](https://duskwatch.me/checklist?ref=github).

Built by the people behind [Duskwatch](https://duskwatch.me), which runs these checks from outside
the instance across several client instances. This workflow is the single-instance version, free
and MIT, and there is nothing in it that phones home — read the JSON.

Tested against n8n 2.37.
