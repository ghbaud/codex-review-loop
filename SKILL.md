---
name: codex-review-loop
description: Use when Claude wants an adversarial review, design critique, plan critique, or second-opinion debugging from Codex AND wants to engage with the findings rather than receive a one-shot report. Sets up an Epic bead, asks Codex to file each finding as a child bead, then runs a back-and-forth where Claude responds to each finding, Codex responds to Claude, and so on until both agree (or the user decides to stop). Trigger this skill whenever you are about to dispatch /codex:codex-rescue for review, critique, design feedback, or a second-opinion pass; whenever the user asks for a "codex review loop", "iterative codex review", "back-and-forth with codex", "argue with codex", or similar; or whenever you want the durability of a written record across rounds. Skip for one-shot reviews where no rebuttal is wanted, for implementation work, or in projects without bd.
---

# Codex Review Loop

This skill runs a structured back-and-forth between Claude and Codex on a review task. Findings live in beads. Each finding has its own thread of comments, prefixed `[claude]` or `[codex]` so the conversation is auditable. The loop continues until every finding is closed by agreement, or until the user steps in to break a deadlock.

**Always use a dedicated review epic.** Even when the work being reviewed already has an implementation epic, the review loop gets its own fresh epic. See Step 1 for the reasoning.

## Status: actively maintained

The protocol mechanics are stable: tested end-to-end against itself three times, then used on two real reviews on 2026-04-27 (WheelHouse Google STT retraction policy review wh-76yv with 5 findings closed across 2 rounds; wh-58vf molecule-structure review with 5 findings closed across 3 rounds). Both real runs converged.

The documentation is NOT stable -- every real run so far has produced at least one meaningful skill change (polling regex, repo-access note, round-limit reframing, queued-job-stuck failure mode, others). Expect that pattern to continue for the next several uses.

If anything in this protocol fails or behaves unexpectedly -- a bd command errors out, Codex misuses a flag, the polling script does not detect a state, the parse misses a section, you find yourself rationalizing a stop, anything -- surface it to the user immediately AND fold the lesson back into this file (Pitfalls or Other failure modes section). Do NOT silently work around the issue. The user is iterating on this skill and needs to see real-world friction so the next revision can address it.

## Why this exists

A single Codex rescue gives you findings without a way to push back. Some findings are valuable, some are wrong, some are based on misreadings. This skill lets Claude examine each finding, push back where appropriate, and let Codex defend or concede. The result is a durable record of what was reviewed, what was decided, and why.

The pattern only pays off when the value is in the dialogue. Use it for adversarial reviews and design critiques. Do not use it for "fix this bug" or "add this feature" -- those are implementation tasks for `/codex:codex-rescue` directly.

## Design rule: symmetric bd ownership

Each speaker writes its own bd output. Codex writes Codex's findings, Codex's response comments, and closes beads when Codex agrees. Claude writes Claude's response comments and closes beads when Claude agrees. Neither side speaks for the other.

This was tested end-to-end. Codex can call `bd create`, `bd comment`, and `bd close` directly without trouble. The earlier "Claude owns all bd" design was an overcorrection based on a Codex stall during one test -- not a real limitation.

## Identity prefixes

Every comment in the loop starts with one of:
- `[claude]` -- written by Claude
- `[codex]` -- written by Codex

The dispatch prompt instructs Codex to prefix every comment with `[codex]`. Claude prefixes its own comments with `[claude]`. The prefix is the only way the audit trail stays readable across rounds. If a Codex comment arrives without the prefix, Claude notes the omission in its next response on that finding but does NOT edit Codex's comment.

## Prerequisites

Before starting, verify:
- Current directory is a bd-tracked project (`.beads/` exists, or a bd-tracked parent does).
- The `codex:codex-rescue` subagent type is available via the Agent tool. Do NOT invoke `/codex:rescue` via the Skill tool -- in testing, the Skill-tool dispatch hung in "Initializing..." indefinitely.
- The user has confirmed they want this loop. Each round costs real Codex tokens; the user should know.

## Dispatch path

Always use the Agent tool with `subagent_type: "codex:codex-rescue"`. Always include `--background --fresh` in the prompt:
- `--background` gives Claude a job ID and a deterministic polling target. The rescue subagent's auto-selection picks background for complex tasks anyway.
- `--fresh` skips the resume-prompt that the rescue agent otherwise asks via AskUserQuestion.

**Two-layer dispatch -- the rescue subagent is NOT the Codex job.** This trips people up; pay attention.

There are two separate background processes:

1. **The rescue subagent** is the wrapper Claude invokes via the Agent tool. Its only job is to dispatch the Codex companion and exit. It usually finishes in 30-90 seconds.
2. **The Codex job** is the actual review work. It runs inside the codex-companion process and is identified by an ID like `task-XXXXX-XXXXX`. It typically takes 3-10 minutes for a real review.

When you call the Agent tool with `run_in_background: true`, you get a "completed" notification when the **rescue subagent** finishes -- which means dispatch is done, NOT that the review is done. The rescue subagent's return value contains a sentence like `Codex Task started in the background as task-XXXXX-XXXXX. Check /codex:status task-XXXXX-XXXXX for progress.` That `task-XXXXX-XXXXX` is the Codex job ID. The actual review work runs against THAT id and must be polled separately via the codex-companion (see Polling pattern below).

**Recommendation: do NOT pass `run_in_background: true` to the Agent tool.** Dispatch is fast (under two minutes), and running the rescue subagent in the foreground means Claude blocks until the Codex job ID is in hand and can immediately start the background Bash poll against it. Background mode on the Agent tool just adds a confusing intermediate "completed" notification that does not mean what it sounds like.

If you DO run the Agent in background (e.g., to free Claude up for other work during dispatch), be explicit in your status messages to the user that the agent's "completed" event is dispatch-only, and you still have to poll the Codex job. The skill's first real adversarial review (2026-04-27, this skill on its own implementation) caught this exact confusion in the user-facing messaging -- Claude said "round 1 is running... I will be notified when it finishes" before the rescue subagent finished, then said "the actual Codex job is still running" after. The user reasonably asked "wait, did it finish or not?" The right phrasing in the dispatch acknowledgement is something like: "Rescue subagent dispatched (will exit in ~1 minute). Codex job ID will arrive then; I'll start polling against it and let you know when the actual review work completes."

The Codex sandbox cannot reliably call `bd show` mid-run. A prefetch hook (`~/.claude/hooks/codex-prefetch-bd.py`) intercepts dispatches that name a bd ID and forces Claude to inline the bead's content. To skip the prefetch when Claude has already inlined everything, include `<bead_context>SKIP</bead_context>` in the prompt.

**Codex CAN read the repo directly.** It has full file-system read access to the project tree. Do NOT inline source code into the dispatch prompt -- give file paths plus line ranges (e.g., `services/wheelhouse/integrations/websocket_manager.py:256-301`) and let Codex open the file. Inline only material Codex cannot otherwise see: bd state (use `<bead_context>SKIP</bead_context>` plus inlined `bd show` output), files outside the repo (e.g., `C:/Users/dhite/Downloads/trace.txt`), conversation context (the current task, the design under review, the constraints). This keeps dispatch prompts short and lets Codex consult the actual code instead of relying on Claude's snapshot, which can drift from the source.

## Wording rules: do not trip the read-only classifier

The rescue subagent strips `--write` from the Codex invocation if the dispatch prompt looks like "review without edits." The relevant rule lives in `~/.claude/plugins/cache/openai-codex/codex/<version>/agents/codex-rescue.md`:

> Default to a write-capable Codex run by adding `--write` unless the user explicitly asks for read-only behavior or only wants review, diagnosis, or research without edits.

With `--write` off, the codex-companion approval policy declines every `bd` invocation -- including read-only ones like `bd show` and `bd --version`. The Codex job log shows lines like `Command declined: ... bd prime (exit -1)`. Codex cannot file findings, post comments, or close beads. The review still produces analysis, but the analysis lands as freeform stdout instead of bd state.

This skill REQUIRES `--write` on. Filing findings is the entire output contract. The dispatch prompt must not phrase the task in a way that strips `--write`.

Trigger phrases the classifier reads as read-only:
- "review-only"
- "do not modify code" / "do not edit files"
- "without edits"
- "code only" / "test code only" (when paired with a "do not modify" framing)
- The `<action_safety>` block the rescue subagent inserts also tends to contain "do not edit" wording when it sees these phrases in the user task.

The dispatch prompt MUST avoid those phrases AND state explicitly that the output is bd writes. Frame the output positively rather than the negative restriction:

> Output: bd writes (`bd create`, `bd comment`, `bd close`). The reviewer files findings as child beads of the epic. The reviewer must not edit source files in the workspace; if a fix is obvious, name it in the finding's body, do not apply it.

That keeps `--write` on (so bd works) AND tells Codex to keep its hands off source. Codex respects the source-files boundary at the prompt level; the sandbox no longer enforces it.

**Verify after dispatch.** Once the rescue subagent returns the Codex job ID, open `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/<job-id>.json`. The `request.write` field must be `true`. If it is `false`, the rescue subagent stripped `--write` and Codex will not be able to write to bd. Cancel the job (`node "$COMPANION" cancel <job-id>`) and re-dispatch with corrected wording. Catching this BEFORE the job runs costs no Codex tokens; catching it AFTER means the round's analysis must be salvaged manually as freeform text.

## Polling: define the companion path once

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

## Polling pattern

Background dispatch returns a job ID immediately. Run the polling script below via the `Bash` tool with `run_in_background: true`. The script polls the Codex companion every 30 seconds and exits on one of three conditions:

1. **Terminal status.** The Codex job reached `completed`, `failed`, `cancelled`, or `interrupted`. Normal completion.
2. **Stale log (running jobs only).** The Codex job log file has not been written to for 5 minutes WHILE status is `running`. This usually means Codex is stuck, but it can also mean deep reasoning. When this fires, ask the user before continuing: "Codex log has been quiet 5 minutes. Status still says running. This can mean stuck or deep reasoning. Continue waiting, or cancel?" The check is gated on `running` so a job that sits in `queued` does not trip stale-log; a stuck queued job has no log writes at all and would otherwise exit at 5 minutes instead of reaching the 15-minute timeout where the queued-stuck guidance applies.
3. **Hard timeout.** 15 minutes elapsed without terminal status or stale log. Ask the user how to proceed -- the job may be genuinely stuck OR may still be doing useful work.
4. **Status parse failure.** Three consecutive polls return no parseable status. Usually means the job ID is wrong, the companion path is wrong, or the companion's status output format changed. Exit early so Claude can investigate instead of waiting 15 minutes.

Claude is notified when the background script exits. The exit reason appears in the final stdout line; Claude reads it from the completed run output. Intermediate status lines are written to the run output but are not delivered as separate notifications.

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

The mtime check must read the Codex job log file only -- not any wrapper output file. Otherwise the heuristic loses meaning.

**Why background `Bash` and not `Monitor`.** `Monitor` delivers each stdout line as a separate notification to Claude. With 30-second polling over a 10-minute review, that is roughly 20 notifications. Each notification is a separate model turn, with its own token and time cost on top of the polling output itself. Even with prompt caching the wake-ups multiply API turns. Background `Bash` produces one completion notification when the script exits. The trade-off: Claude cannot see status during the run -- if the user wants to cancel mid-run for any reason other than a stale log, they have to interrupt the session.

A `queued` status that persists past the 15-minute hard-stop means the Codex companion's queue dispatcher is stuck (observed 2026-04-27: a job sat in `queued` state for 17 minutes with no other active jobs blocking it). If you see `status=queued elapsed=15m` in the timeout message, the cancel-and-redispatch path may not work either -- the companion's `cancel` invokes `taskkill` against a wrapper PID, and on Git Bash that fails with a path-mangling error AND leaves the queued job intact. In that case the cleanest move is to dispatch a fresh round-N job; the stuck job stays harmlessly queued and the new one runs. Surface the issue to the user before retrying so they know the spend on the stuck job is wasted.

## Project-specific review criteria

Some projects have standing concerns that every review should evaluate. Those concerns live in a project-level skill named `project-review-criteria`, located at `<project>/.claude/skills/project-review-criteria/SKILL.md`.

Before dispatching round 1, check the available-skills list for a skill named `project-review-criteria`. If it is present, invoke it via the Skill tool. The skill body lists project-specific items to add to the dispatch prompt under "What to look for" and, where relevant, "Constraints (do not flag)". Inline those items verbatim into the round-1 prompt.

If no such skill is registered, skip this step. Do not invent project criteria.

The same project criteria apply to every round of the loop, but only the round-1 dispatch needs to inline them. Codex carries the criteria forward implicitly via the findings it filed in round 1.

## Protocol

### Step 1: Create the review epic

**Always create a NEW, dedicated review epic. Never reuse an existing implementation epic, even if the work being reviewed already has one.**

The review epic is for review findings only -- one finding per child bead, written by Codex, responded to by Claude, and closed when both sides converge. The implementation epic (the one that tracks the code work itself) is a different concept and lives separately. Reusing the implementation epic mixes review findings into the implementation epic's children list and confuses the audit trail in both directions:
- The implementation epic's "open children" no longer reflects implementation work left to do; it includes review findings that may already be answered by the existing implementation children.
- The review loop's termination check (close the epic when all children are closed) cannot fire, because the implementation children are still open.

If the work being reviewed has its own implementation epic, link the new review epic to it from the description ("Review of design proposed under `<impl-epic-id>` (`<impl-epic-title>`)") rather than nesting one under the other.

```
bd create --type=epic --priority=2 --label=review-loop \
  --title="<review subject>" --description="<full context>"
```

Description should include what is being reviewed (file paths, plan documents, code branches), what kind of review is wanted, constraints the reviewer should not flag, and a pointer to any related implementation epic. Capture the epic ID.

### Step 2: Dispatch Codex round 1

Read the review target into Claude's context first so Claude can inline it. Codex cannot reliably call `bd show` mid-run, so the dispatch prompt must contain everything Codex needs.

**Inferring constraints from conversation context.** When the user invokes this skill from a session where you have already been working a problem together, you have implicit context about what the user has considered and ruled out. Mine that context for the "Constraints (do not flag)" block instead of leaving it empty or asking the user to enumerate everything. Concrete things to extract:

- Design choices the user has explicitly settled ("we are using SharedMemory, not a queue").
- Tradeoffs the user has acknowledged and accepted ("yes the polling adds latency, that is fine").
- Scope exclusions ("the prefetch hook is out of scope for this review").
- Things that look problematic but have a reason ("the comment says TODO but we are intentionally deferring").
- Approaches the user has tried and rejected ("we tried X, it broke Y, do not propose X again").

If you have NO prior context (the user invoked this skill without having worked the problem with you), ask the user for constraints before dispatching. Better to spend one user turn than to burn Codex tokens on settled questions.

Use the Agent tool with `subagent_type: "codex:codex-rescue"`. The prompt (read the "Wording rules" section above before composing -- the Output and source-files lines are NOT optional):

```
--background --fresh

Adversarial review for epic <epic-id>.

What to review:
<inline file content, plan text, or excerpt>

What to look for:
<concrete list -- design flaws, race conditions, missing edge cases, etc.>

Constraints (do not flag):
<list anything Codex should ignore>

Output: bd writes only. The reviewer files findings as child beads of
<epic-id> via `bd create`, posts the round-complete comment via
`bd comment`, and uses `bd close` if any finding is unambiguous. The
reviewer must not edit source files in the workspace; if a fix is
obvious, name the fix in the finding's body, do not apply it.

For each distinct finding, file a child bead of <epic-id>:
  bd create --type=task --parent=<epic-id> --priority=2 \
    --label=review-loop --label=review-loop-finding \
    --title="<short summary>" --body-file <tmp-file>

IMPORTANT label syntax: pass each label as a separate --label flag. Do NOT use
"--label X Y" with a space -- bd reads that as one label "X Y".

Use --body-file (not --description) for the description text, since findings
contain backticks, hyphens, and quotes. Write the body to a temp file first.

When you have filed all findings, post a single completion comment on
<epic-id>:
  bd comment <epic-id> "[codex] Round 1 complete. Filed N findings: <id1>, <id2>, ..."

If you find no issues, post:
  bd comment <epic-id> "[codex] Round 1 complete. No findings."

Prefix every comment with [codex].
```

**Do NOT use the words "review-only", "do not modify code", "without edits", or "code only" anywhere in the dispatch prompt.** They look harmless but they trip the rescue subagent's classifier and strip `--write`. See the Wording rules section above for the rule and the verification step.

Capture the **Codex job ID** from the rescue subagent's return value -- the `task-XXXXX-XXXXX` string in the wrapper's "Codex Task started in the background as ..." message. Do NOT use the Agent tool's internal agent ID; that is a different identifier and the codex-companion does not recognise it. Run the polling loop above against the Codex job ID.

### Step 3: Verify what Codex did

Once status is `completed`, run:
```bash
bd children <epic-id>
bd comments <epic-id>
```

Verify each new child bead has:
- The two labels `review-loop` and `review-loop-finding` (not a phantom merged label)
- A `[codex]` prefix in the description
- Sensible title and body

If any are malformed, fix with `bd update` or note the discrepancy in your response.

If Codex returned "No findings" but Claude expected some, present the situation to the user: "Codex found no issues with X. Close the epic, or do you want me to dispatch a re-review with a sharper prompt?" Do NOT close the epic automatically.

### Step 4: Process each finding (Claude's responses)

For each open child bead, decide one of these and respond directly via `bd comment`. Most responses are plain prose and use the inline form:

```bash
bd comment <finding-id> '[claude] Agree. <action: will fix in this PR / filed follow-up bead <id> / already fixed in <ref>>.'
bd close <finding-id>
```

Single-quoted args are safe for any text without literal single quotes. If the response contains backticks, hyphens at the start of lines, or other shell metacharacters, switch to stdin via heredoc:

```bash
bd comment <finding-id> --stdin <<'EOF'
[claude] <multi-line response with `code` and special chars>
EOF
```

The `<<'EOF'` (single-quoted) form prevents any shell interpretation of the body. Do NOT default to the most defensive option uniformly -- triage on actual content. Most agreement comments are plain prose and the inline form is faster.

Decision options:
- **Agree.** Comment + close.
- **Counter-propose.** Comment with reasoning and an alternative. Leave open.
- **Reject.** Comment with reasoning for why this is not a problem. Leave open.
- **Need more information.** Comment with a specific question. Leave open.

### Step 5: Dispatch Codex round 2 (and onward)

Gather the IDs of every still-open finding. Codex cannot reliably read them via `bd show`, so Claude inlines the full state. Build a `<bead_context>SKIP</bead_context>` block (to bypass the prefetch hook, which would otherwise re-inject what Claude already has) plus the finding contents:

```bash
{
  for bid in <epic-id> <finding-id-1> <finding-id-2> ...; do
    echo "--- bd show $bid ---"; bd show $bid; echo ""
  done
} > /c/tmp/r2-context.txt
```

Note: `/c/tmp/` is the Git Bash form of `C:\tmp\`. The Write tool's `/tmp/` ALSO maps to `C:\tmp\` (not the MSYS `/tmp/` which is `%TEMP%`). Be consistent or paths mismatch. Use `C:/tmp/` in any path that crosses between the Write tool and a Windows-native binary like `bd`.

Dispatch round 2:

```
--background --fresh

<bead_context>SKIP</bead_context>

Round <N> for epic <epic-id>. The findings listed below are still open. For each one, read Claude's most recent [claude] comment and respond.

For each finding:
  - If you agree with Claude's pushback or accept the counter-proposal:
      bd comment <finding-id> "[codex] <agreement text>"
      bd close <finding-id>
  - If you maintain your position:
      bd comment <finding-id> "[codex] <new reasoning, possibly revised counter-proposal>"
      (leave open)
  - If Claude asked a question:
      bd comment <finding-id> "[codex] <answer>"
      Close if your answer fully resolves the issue; otherwise leave open.

When done, post a single completion comment on <epic-id>:
  bd comment <epic-id> "[codex] Round <N> complete. <per-finding summary>"

Prefix every comment with [codex]. Use --body-file or --stdin if your comment
contains backticks or other shell metacharacters.

<inlined bd show output for epic and every open finding here>
```

Then run the polling loop again. After completion, verify state with `bd children <epic-id>` and `bd comments <finding-id>` for each finding that should have moved.

### Step 6: Round-limit check

**Default is to keep running rounds, not to stop.** Do not stop between rounds unless you have an excellent reason that genuinely needs the user's input. Laziness is not an excellent reason. "I anticipate the threshold might fire next round" is not an excellent reason. "The next move is obvious and Codex named it" is not a reason to stop -- it is a reason to make the move and continue.

The legitimate stop conditions are:

- **Genuine deadlock.** Both sides have re-stated their positions without movement for 2+ rounds and neither side concedes. The user needs to break the tie.
- **Scope explosion.** Codex's pushback would require expanding the work beyond what the user authorized for this loop (e.g., a finding that says "this whole subsystem needs rewriting"). The user needs to confirm the bigger scope before more Codex tokens go into it.
- **Externally blocked.** A finding's resolution depends on something the user controls and Claude cannot decide alone -- e.g., a product decision, a budget call, a dependency on another team's work.
- **Hard limit reached.** A still-open finding has accumulated 4+ comments AND the conversation is not visibly converging (each round adds new disagreement rather than narrowing it). At that point ask the user how to proceed.

When you DO stop, present the four options below. When you do NOT stop -- the common case -- pick the right option yourself and execute:

1. Continue iterating -- dispatch Codex for another round.
2. Accept Claude's position -- Claude documents the conclusion, closes the finding.
3. Accept Codex's position -- Claude implements the proposed fix or files a follow-up bead, then closes.
4. Document as known disagreement -- Claude posts a final summary comment, closes the finding.

If the user picks "continue iterating" (or you decide to continue past the 4-comment hard-limit yourself with explicit reasoning), post a marker comment on the finding: `[claude] Round-limit reset at comment <N>`. Future round-limit checks count only comments posted after the latest reset marker, so the warning does not fire every round on long discussions.

**Anti-pattern observed 2026-04-27:** after round 2 of the wh-58vf molecule review, finding wh-58vf.5 had 3 comments and was open with an obvious next move (option 3 -- accept Codex's revised proposal and encode it). Claude stopped anyway and presented the four options to the user, framing it as "one more round before the round-limit check fires". The user pushed back: "why not option C". Lesson: do not pre-empt the threshold. The threshold IS the threshold; below it, keep moving.

### Step 7: Termination and summary

When all child beads are closed:
1. Post a final summary on the epic: `bd comment <epic-id> '[claude] Loop complete. <N> findings: <X> agreed, <Y> rejected with reasoning, <Z> resolved by user decision.'`
2. Close the epic: `bd close <epic-id>`.
3. Report to the user inline: list each finding with its short title, the resolution, and any follow-up beads created.

## Discoveries during the loop

If Claude or Codex spots something new while working on a finding:

- **Discovery about an existing finding** (the proposed fix also breaks something else, or the analysis was incomplete): post as an additional comment on that finding's bead. Keep related discussion together.
- **Discovery completely unrelated to any finding** (a separate issue spotted while tracing code): file a new bead with NO parent and a `discovered-during-review` label. Mention it in the loop's final summary. Do NOT add it as a child of the review epic -- the loop completes when all child beads are closed, and a new child added mid-loop pushes the goalposts.
- **Discovery that should have been a round-1 finding** (Codex missed it; Claude finds it now): file as a child of the review epic with the standard `review-loop-finding` label so the audit trail is complete. The user gets it surfaced in the next round's dispatch or in the final summary.

For Codex specifically: Codex is in a constrained dispatch and should not freelance new beads outside the prompt's scope. The right place for a Codex discovery is a note inside the relevant `[codex]` comment ("Note: while reviewing X, I noticed Y is similarly affected -- recommend filing separately"). Claude then decides whether to file a new bead.

## Verifying Codex's writes

After every Codex round, verify the bd state directly. Codex is good but not perfect. Common issues to check for:

- **Phantom labels** like `review-loop review-loop-finding` (a single label with a space). Codex sometimes runs `--label "X Y"` instead of `--label X --label Y`. Fix with `bd update <id> --remove-label "X Y" --add-label X --add-label Y`.
- **Missing `[codex]` prefix.** Note in Claude's next response; do not edit Codex's comment.
- **Comments without a `bd close`** when Codex said it agreed. Close on Codex's behalf and add a `[claude]` follow-up noting the action.
- **Beads in the wrong project.** bd walks up the filesystem to find `.beads/`. Verify new bead IDs have the expected prefix (e.g., `wh-` for WheelHouse).

**Source-tree check after every round.** Once `--write` is on (the rule for this skill), Codex has the technical ability to edit files in the workspace. The dispatch prompt's "do not edit source files" instruction is what holds Codex back; it is behavioral discipline, not a sandbox guarantee. Run a tracked-tree check after every round and BEFORE responding to findings:

```bash
git status --porcelain
git diff --stat
```

If either reports any file change Claude did not initiate, abort the loop and ask the user before proceeding. A common false-positive: project files generated by hooks (file maps, knowledge-bead exports) that the SessionStart or PreToolUse hooks regenerate. Filter those out by name; flag anything else.

**JSON write-flag check after dispatch.** Open the job's metadata file at `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/<job-id>.json`. The `request.write` field must be `true`. If it is `false`, see "Wording rules" -- the rescue subagent stripped `--write` and Codex cannot file findings. Cancel and re-dispatch.

## Cancelling a stuck job

1. Try the companion's cancel: `node "$COMPANION" cancel <job-id>`. On Windows / Git Bash this may fail with a `taskkill` path-mangling error.
2. If cancel fails AND the user explicitly confirms the broad action (this kills ALL active Codex jobs in the workspace, not just yours): list the codex backend processes with `Get-Process | Where-Object { $_.ProcessName -in @('codex','Codex') } | Select-Object Id, ProcessName, StartTime, CPU`, present them to the user with start times so the user can confirm they are stale, then `Stop-Process -Id <pid> -Force`.
3. The companion's job-state file may still show `running` after the kill. This is cosmetic only -- the actual processes are gone. Do NOT edit the JSON state file by hand; it is internal codex state and the edit may not propagate.

After cancel, either reduce the scope of the review and redispatch, or abandon the loop.

## Other failure modes

**Codex returns no findings (Codex-side reports "No findings").** Trust Codex unless the user disagrees. Report to the user: "Codex found no issues with X. Close the epic, or do you want me to dispatch a re-review with a sharper prompt?" Do not close automatically.

**Codex returns malformed beads (e.g., garbled descriptions, missing fields).** Treat each one individually. Either fix with `bd update` or close it as malformed and dispatch a re-review for that specific finding.

**Codex decided to write a freeform stdout summary instead of filing beads.** Most common root cause: the rescue subagent stripped `--write` from the Codex invocation because the dispatch prompt looked like "review without edits" -- see the Wording rules section for triggers and the verification step that catches this BEFORE the job runs. Symptom in the job log: every `bd` invocation shows `Command declined: ... bd <subcommand> (exit -1)`. The job's JSON metadata at `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/<job-id>.json` shows `"write": false`.

If you catch the misclassification before the job runs, cancel and re-dispatch with corrected wording. If the job already completed: the Codex result fetch via `node "$COMPANION" result <job-id>` shows the freeform text. Salvage manually: read the text, file beads from it as if Claude were the author, prefix descriptions with `[codex] originally posted in chat without filing as bead:` so the audit trail is honest. Real example: the wh-pbpy run on 2026-04-28 hit this exact path; 6 findings were salvaged from the freeform summary and the loop continued normally from round 2.

**Codex run exceeds the 15-minute polling window.** Status is `TIMEOUT_15MIN`. The job may be genuinely stuck OR may still be doing useful work. Ask the user before cancelling. If the user confirms the job is stuck (or it has been queued the entire time -- see the queued-stuck note in the polling section), cancel per the section above and reduce scope.

**Foreground vs background mismatch.** The rescue subagent's auto-selection prefers background for complex tasks, even if the prompt did not explicitly choose. Always include `--background` explicitly in the prompt to avoid surprise.

## Worked example

User asks Claude to get an adversarial review of a draft plan at `docs/plans/2026-04-27-foo.md`.

1. Claude creates `wh-abc1` (Adversarial review of foo plan), an epic.
2. Claude reads the plan file. Claude dispatches Codex with the round-1 template, inlining the plan content and naming `wh-abc1`. Background mode, fresh thread.
3. Claude starts the polling script via `Bash` with `run_in_background: true`. After 4 minutes, the script exits with `TERMINAL: completed`.
4. Claude verifies: `bd children wh-abc1` shows three new findings `wh-abc1.1`, `wh-abc1.2`, `wh-abc1.3`. Each has the right labels and a `[codex]` description prefix.
5. Claude processes each:
   - `wh-abc1.1` (concurrency race): Claude agrees, files follow-up bead `wh-abc2` with no parent (`discovered-during-review` label), comments `[claude] Agree. Filed wh-abc2 for the implementation.`, closes.
   - `wh-abc1.2` (suggested abstraction is premature): Claude rejects with reasoning, leaves open.
   - `wh-abc1.3` (asks why a config knob exists): Claude answers, leaves open.
6. Claude builds the round-2 dispatch with `wh-abc1.2` and `wh-abc1.3` inlined, dispatches with `<bead_context>SKIP</bead_context>`.
7. Codex returns. `wh-abc1.2` -- Codex concedes, posts `[codex] Agree, your reasoning about premature abstraction holds.`, closes. `wh-abc1.3` -- Codex acknowledges the answer, closes.
8. All children closed. Claude posts a summary on `wh-abc1`, closes the epic. Reports to user.

## When to skip this skill

- One-shot review with no expected pushback: use `/codex:codex-rescue` directly.
- Implementation tasks (write code, fix a bug, refactor): use `/codex:codex-rescue` directly.
- Project has no bd tracking: the skill cannot persist state.
- The user wants a quick verbal opinion, not a written record.
