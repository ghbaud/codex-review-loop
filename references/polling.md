# Polling pattern

Background dispatch returns a Codex job ID immediately. Run the polling script below via the `Bash` tool with `run_in_background: true`. The script polls the Codex companion every 30 seconds and exits on one of four conditions:

1. **Terminal status.** The Codex job reached `completed`, `failed`, `cancelled`, or `interrupted`. Normal completion.
2. **Stale log (running jobs only).** The Codex job log file has not been written to for 5 minutes WHILE status is `running`. This usually means Codex is stuck, but it can also mean deep reasoning. When this fires, ask the user before continuing: "Codex log has been quiet 5 minutes. Status still says running. This can mean stuck or deep reasoning. Continue waiting, or cancel?" The check is gated on `running` so a job that sits in `queued` does not trip stale-log; a stuck queued job has no log writes at all and would otherwise exit at 5 minutes instead of reaching the 15-minute hard timeout where the queued-stuck guidance applies.
3. **Hard timeout.** 15 minutes elapsed without terminal status or stale log. Ask the user how to proceed -- the job may be genuinely stuck OR may still be doing useful work.
4. **Status parse failure.** Three consecutive polls return no parseable status. Usually means the job ID is wrong, the companion path is wrong, or the companion's status output format changed. Exit early so Claude can investigate instead of waiting 15 minutes.

Claude is notified when the background script exits. The exit reason appears in the final stdout line; Claude reads it from the completed run output. Intermediate status lines are written to the run output but are not delivered as separate notifications.

## Companion path

Define this at the top of any Bash that calls the codex companion, so `status`, `result`, and `cancel` always use the same path:

```bash
# Default path includes plugin version. Update if version changes.
COMPANION="${CLAUDE_PLUGIN_ROOT:-C:/Users/dhite/.claude/plugins/cache/openai-codex/codex/1.0.3}/scripts/codex-companion.mjs"
```

PowerShell equivalent:
```powershell
$COMPANION = if ($env:CLAUDE_PLUGIN_ROOT) { "$env:CLAUDE_PLUGIN_ROOT/scripts/codex-companion.mjs" } else { "C:/Users/dhite/.claude/plugins/cache/openai-codex/codex/1.0.3/scripts/codex-companion.mjs" }
```

Then use `node "$COMPANION" status <id>`, `node "$COMPANION" result <id>`, and `node "$COMPANION" cancel <id>` consistently.

## Polling script

```bash
JOB=<job-id>
COMPANION="${CLAUDE_PLUGIN_ROOT:-C:/Users/dhite/.claude/plugins/cache/openai-codex/codex/1.0.3}/scripts/codex-companion.mjs"
START=$(date +%s)
empty_count=0
while true; do
  s=$(node "$COMPANION" status "$JOB" 2>/dev/null)
  status=$(echo "$s" | grep -oE '\| (queued|pending|running|completed|failed|cancelled|interrupted) \|' | head -1 | tr -d ' |')
  elapsed=$(echo "$s" | grep -E '^  Elapsed:' | head -1 | sed 's/.*Elapsed: //')
  logfile=$(echo "$s" | grep -E '^  Log:' | head -1 | sed 's/.*Log: //' | tr -d '\r')
  log_age="?"
  if [ -n "$logfile" ] && [ -f "$logfile" ]; then
    log_mtime=$(stat -c %Y "$logfile" 2>/dev/null || echo 0)
    log_age=$(( $(date +%s) - log_mtime ))
  fi
  echo "$(date +%H:%M:%S) status=$status elapsed=$elapsed log_age=${log_age}s"
  if [ -z "$status" ]; then
    empty_count=$((empty_count + 1))
    if [ "$empty_count" -ge 3 ]; then
      echo "STATUS_ERROR: companion returned no parseable status 3 times -- check job ID and companion path"; exit 0
    fi
    sleep 30
    continue
  fi
  empty_count=0
  case "$status" in
    completed|failed|cancelled|interrupted) echo "TERMINAL: $status"; exit 0 ;;
  esac
  if [ "$status" = "running" ] && [ "$log_age" != "?" ] && [ "$log_age" -gt 300 ]; then
    echo "STALE_LOG: ${log_age}s -- Codex log quiet, ask user"; exit 0
  fi
  if [ $(( $(date +%s) - START )) -gt 900 ]; then
    echo "TIMEOUT_15MIN: ask user"; exit 0
  fi
  sleep 30
done
```

The mtime check must read the Codex job log file only -- not any wrapper output file. Otherwise the stale-log check no longer detects whether Codex itself is making progress.

## Why background `Bash` and not `Monitor`

`Monitor` delivers each stdout line as a separate notification to Claude. With 30-second polling over a 10-minute review, that is roughly 20 notifications. Each notification is a separate model turn, with its own token and time cost on top of the polling output itself. Even with prompt caching, each notification still triggers a new turn. Background `Bash` produces one completion notification when the script exits. The trade-off: Claude cannot see status during the run -- if the user wants to cancel mid-run for any reason other than a stale log, they have to interrupt the session.
