# Cancelling a stuck Codex job

1. Try the companion's cancel: `node "$COMPANION" cancel <job-id>`. On Windows / Git Bash this may fail with a `taskkill` path-mangling error. (See `references/polling.md` for the `$COMPANION` definition.)
2. If cancel fails AND the user explicitly confirms the broad action (this kills ALL active Codex jobs in the workspace, not just yours): list the codex backend processes with `Get-Process | Where-Object { $_.ProcessName -in @('codex','Codex') } | Select-Object Id, ProcessName, StartTime, CPU`, present them to the user with start times so the user can confirm they are stale, then `Stop-Process -Id <pid> -Force`.
3. The companion's job-state file may still show `running` after the kill. This is cosmetic only -- the actual processes are gone. Do NOT edit the JSON state file by hand; it is internal codex state and the edit may not propagate.

After cancel, either reduce the scope of the review and redispatch, or abandon the loop.

## Stuck-queued jobs

A `queued` status that persists past the 15-minute hard-stop means the Codex companion's queue dispatcher is stuck. The companion's `cancel` invokes `taskkill` against a wrapper PID, and on Git Bash that fails with a path-mangling error AND leaves the queued job intact. In that case the cleanest move is to dispatch a fresh round-N job; the stuck job stays harmlessly queued and the new one runs. Surface the issue to the user before retrying so they know the spend on the stuck job is wasted.
