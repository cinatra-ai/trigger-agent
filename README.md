# Trigger Agent

Decide when a workflow actually runs. This agent gives you a small picker — run now, run once at a specific time, or run on a recurring schedule — then asks you to confirm before the trigger is saved. It is what every long-running workflow uses to capture its "when" before its "what" starts.

## Capabilities

- Pick between immediate, one-time scheduled, and recurring triggers
- Capture a date, time, and timezone for a scheduled run
- Capture a recurring schedule from a plain-language prompt or cron expression
- Show a read-only summary and wait for explicit confirmation
- Persist the confirmed trigger to the parent workflow run
