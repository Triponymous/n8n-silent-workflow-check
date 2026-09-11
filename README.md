# Silent workflow check for n8n

Finds scheduled n8n workflows that stopped running.

That failure produces no failed execution, so an error workflow never fires for it. It happens
after a restart that did not re-register the schedule triggers, or when someone switches a
workflow off while debugging and forgets it. The execution list looks clean and nothing has run
for days.

## What it does

Reads your own instance through the n8n API. For every active workflow that starts itself
(Schedule, Cron or Interval trigger) it looks up the last run and compares it with that
workflow's own schedule. A workflow counts as silent once twice its interval plus five minutes
has passed without a run: a 15-minute sync is flagged after 35 minutes, a daily report after two
days, and there is no threshold to set per workflow.

Workflows started from outside (webhooks, forms, chats, app triggers) are not judged. Idle is
normal for them, and a check that calls idle broken gets muted within a week.

```
Every morning → Configuration → Get workflows → One item per workflow → Get last run
  → Find silent workflows → Heartbeat URL set → Ping heartbeat
```

It always returns exactly one item:

```json
{
  "silent": 1,
  "summary": "1 of 9 checked workflows missed their schedule: Nightly invoice sync (last run 74 h ago, expected every day). 1 could not be judged, see notCovered.",
  "findings": [
    {
      "workflow": "Nightly invoice sync",
      "workflowId": "aBcD1234",
      "expected": "every day",
      "lastRun": "2026-09-08T02:00:11.000Z",
      "hoursSilent": 74,
      "note": "missed at least two scheduled runs"
    }
  ],
  "notCovered": [
    {
      "workflow": "Weekday report",
      "workflowId": "eFgH5678",
      "reason": "Its schedule is not a fixed interval, weekdays only for example, so there is no gap to measure against."
    }
  ],
  "judged": 9,
  "notScheduled": 4,
  "checkSuspect": false,
  "hint": ""
}
```

Put an IF on `silent` greater than 0 after **Find silent workflows** and send `summary` to
wherever your alerts go.

## Setup

1. In n8n: **Settings → n8n API → Create an API key**.
2. Import `silent-workflow-check.json` via **Workflows → Import from File**.
3. On **Get workflows**, create a *Header Auth* credential:
   - Name: `X-N8N-API-KEY`
   - Value: your API key

   Use the same credential on **Get last run**.
4. In **Configuration**, set `baseUrl` to your n8n address without a trailing slash.
5. Optional, see below: set `heartbeatUrl`.
6. Attach your alert node and activate the workflow.

## Watching the watcher

This workflow lives inside the instance it watches. If that instance is down, mid-restart, or its
schedule triggers never re-registered, this check does not run either, and its silence looks
exactly like everything being fine.

The heartbeat closes that gap. Create a free check at a dead man's switch service such as
[healthchecks.io](https://healthchecks.io), give it a period of one day, and paste its ping URL
into `heartbeatUrl`. The workflow pings it at the end of every run, whether or not it found
anything. When the pings stop, the service tells you. The ping node continues on error, so a
heartbeat service that is down never swallows a real finding.

## When it refuses to answer

Some workflows cannot be judged from execution history. The check lists them under `notCovered`
with the reason instead of guessing:

- schedules that are not a fixed interval, weekdays only for example;
- workflows set not to save successful executions: n8n deletes their healthy runs, so they would
  always look silent;
- workflows changed recently that have not run since;
- rare schedules with no saved run, where n8n may already have pruned the last one. A pruned
  execution is not evidence that something did not happen.

If half or more of the checked workflows look silent at once, `checkSuspect` is true and `hint`
says why: either they really all stopped, which is what a restart that did not re-register the
triggers looks like, or the check cannot see what it needs, most often an instance running with
`EXECUTIONS_DATA_SAVE_ON_SUCCESS=none`. Open one of them in n8n before acting on it.

## Known limits

- **It lives inside the instance it watches.** The heartbeat tells you the checker stopped, not
  which of your workflows did.
- **It judges schedules, not work.** A workflow that runs on time and produces nothing looks
  healthy to it.
- **It reads up to 250 active workflows**, one API page. Past that, some are not checked.
- The code nodes avoid arrow functions, template literals and optional chaining on purpose: the
  firewall in front of the n8n template portal rejects uploads that contain them.

## Credit

The heartbeat, the not-covered answers and the hit-rate warning came from posts by
[moneywithjjcom](https://community.n8n.io/u/moneywithjjcom) on the n8n community forum.

## Six more failures that produce no error

Stalled schedules are one of seven. The others: downstream rate limits, credentials that expired,
webhooks nobody calls any more, workflows switched off during debugging, execution history pruned
by `EXECUTIONS_DATA_MAX_AGE`, and daylight saving shifting a schedule by an hour twice a year.

Written up with the one check that catches each, at
[duskwatch.me/checklist](https://duskwatch.me/checklist?ref=github).

Built by the people behind [Duskwatch](https://duskwatch.me), which runs these checks from outside
the instance across every client instance you manage. This workflow is the single-instance
version, free and MIT, and there is nothing in it that phones home. Read the JSON.

Tested against n8n 2.37.
