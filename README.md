# codex-review-loop

A Claude Code skill that runs an iterative, written argument between Claude
and Codex (GPT-5) on a review task. Each finding becomes a tracked issue in
[beads](https://github.com/steveyegge/beads). Each round of back-and-forth is
recorded as a `[claude]` or `[codex]` prefixed comment on that issue. The loop
runs until every finding is closed by agreement or until the user breaks a
deadlock.

## Why

A single Codex review gives you findings without a way to push back. Some
findings are sharp, some are wrong, some are based on Codex misreading the
code. This skill lets Claude examine each finding, push back where
appropriate, and let Codex defend or concede. The result is a durable record
of what was reviewed, what was decided, and why.

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) CLI.
- The [openai-codex plugin](https://github.com/openai/codex) for Claude Code,
  set up via `/codex:setup`. The skill dispatches Codex through
  `subagent_type: "codex:codex-rescue"` and uses the plugin's
  `codex-companion.mjs` for status polling.
- [beads](https://github.com/steveyegge/beads) (`bd`) installed and
  initialized in the project you want to review. The skill stores findings
  and the audit trail in `bd`. Without `bd` the skill cannot persist state.

## Install

Copy the `SKILL.md` into your user skills directory:

```bash
mkdir -p ~/.claude/skills/codex-review-loop
cp SKILL.md ~/.claude/skills/codex-review-loop/SKILL.md
```

Restart Claude Code so the skill is registered.

## Use

In a Claude Code session, ask for an iterative Codex review. Phrases that
trigger the skill:

- "run a codex review loop on this plan"
- "iterative codex review of `path/to/file.py`"
- "argue with codex about this design"
- "back-and-forth with codex on the plan"

The skill takes Claude through:

1. Create a review epic in `bd` with the file paths, branch, or plan being
   reviewed.
2. Dispatch Codex round 1 via `--background --fresh`. Codex files each
   finding as a child issue under the epic.
3. Claude reads each finding and posts a `[claude]` reply: agree (and
   close), counter-propose, reject with reasoning, or ask a question.
4. Dispatch Codex round 2 with the open findings inlined. Codex either
   concedes (closes) or defends (leaves open).
5. Repeat until every finding is closed by agreement, or until the user
   chooses to stop.
6. Post a summary on the epic and close it.

## Notes

- The skill is still under active iteration. The protocol mechanics are
  stable, but the documentation changes after most real runs as new failure
  modes surface. Check the `## Pitfalls` and `## Other failure modes`
  sections in `SKILL.md` if something behaves unexpectedly.
- Codex has filesystem read access to the repo. Pass file paths and line
  ranges in dispatch prompts instead of inlining source code. Inline only
  things Codex cannot otherwise see (`bd` state, files outside the repo,
  conversation context).
- Each round costs real Codex tokens. The skill tells Claude to confirm with
  the user before the first dispatch.

## Optional companion hook

If you frequently dispatch Codex with `bd` IDs in the prompt, an optional
prefetch hook at `~/.claude/hooks/codex-prefetch-bd.py` can intercept the
dispatch and force Claude to inline the bead state automatically. The skill
documents the `<bead_context>SKIP</bead_context>` marker to bypass that hook
when Claude has already inlined everything. The hook is not required; the
skill works without it.

## License

MIT. See `LICENSE`.
