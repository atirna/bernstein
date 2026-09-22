## A host resource outage no longer quarantines tasks

When the spawner refused to start an agent because the host was out of
resources (disk below the spawn floor, memory, file descriptors), the task
burned its respawn and retry budget, and retry exhaustion recorded the task's
title in the cross-run quarantine with `action="skip"`. Quarantine entries are
keyed by task title and persist across runs (they expire after 7 days and
`bernstein quarantine clear` can remove them), so every task that hit the
outage stayed skipped until then — in the reported run, the resource was back
twelve minutes before the run ended, and the tasks remained quarantined
anyway.

The spawn loop now classifies the exception it caught (the spawner's own
`Disk space critical` refusal and the POSIX exhaustion strings it can surface
as) at give-up time and marks the tasks it gave up excused in the quarantine
store; `record_failure` refuses a failure carrying a current excused marker,
so a later task under the same title that fails for its own reasons still
counts. The
exemption is decided from the orchestrator's own observation, not from
`task.result_summary` — agents write that field through the same
`POST /tasks/{id}/fail` endpoint, so a reason an agent authored cannot exempt
a task. The condition recovers
with the machine; a task that failed for its own reasons still quarantines
exactly as before, and retry exhaustion remains terminal for the lineage
within the run. Existing entries from a past outage can be cleared with
`bernstein quarantine clear` (#5963).
