# Wording rules: do not trip the read-only classifier

The rescue subagent strips `--write` from the Codex invocation if the dispatch prompt looks like "review without edits." The relevant rule lives in `~/.claude/plugins/cache/openai-codex/codex/<version>/agents/codex-rescue.md`:

> Default to a write-capable Codex run by adding `--write` unless the user explicitly asks for read-only behavior or only wants review, diagnosis, or research without edits.

With `--write` off, the codex-companion approval policy declines every `bd` invocation -- including read-only ones like `bd show` and `bd --version`. The Codex job log shows lines like `Command declined: ... bd prime (exit -1)`. Codex cannot file findings, post comments, or close beads. The review still produces analysis, but the analysis lands as freeform stdout instead of bd state.

This skill REQUIRES `--write` on. Filing findings is the entire output contract. The dispatch prompt must not phrase the task in a way that strips `--write`.

## The classifier reads the wrapper only

The classifier reads the WRAPPER prompt only. It does not read the staged file the wrapper points to. So restrictive editing language must NEVER appear in the wrapper, even with positive framing around it. Restrictive language belongs in the staged file, which Codex reads after `--write` has been decided.

## Trigger phrases

Any of these in the wrapper, paired with words like "review" or "audit", strips `--write`:

- "review-only"
- "do not modify code" / "do not modify source" / "do not edit files" / "do not edit source"
- "without edits" / "without modifications"
- "code only" / "test code only" (when paired with a "do not modify" framing)
- "must not edit" / "must not modify" / "should not modify" / "should not edit"
- "do not apply" (paired with "fix" or "change")
- "no source changes" / "no file changes"

The `<action_safety>` block the rescue subagent inserts also tends to contain "do not edit" wording when it sees these phrases in the user task, which compounds the problem.

## Wrapper template

The wrapper MUST avoid all of those phrases AND state the output positively. Use this template verbatim:

```
--background --fresh

<bead_context>SKIP</bead_context>

Read C:/tmp/codex-round-<N>-prompt.md in full and act on the dispatch described
in that file. Epic ID is <epic-id>. Output: bd writes -- file each finding as
a child bead of <epic-id> via `bd create`, post round-complete comment via
`bd comment`, close beads with `bd close` as appropriate. Do not summarize in
chat; file findings as beads.
```

Nothing about edits, modifications, or source files. The wrapper carries only: location of the staged dispatch, the epic ID, the output is bd writes, "file findings as beads" not as freeform text.

## Where the source-files boundary goes

Codex needs to know it should not edit source files in the workspace, since `--write` lets it. Put that instruction in the STAGED FILE under the "Output contract" section. Example for the staged file:

> Output contract: bd writes only. The reviewer files findings as child beads of the epic via `bd create`, posts the round-complete comment via `bd comment`, and uses `bd close` if a finding is unambiguous. The reviewer must not edit source files in the workspace; if a fix is obvious, name the fix in the finding's body, do not apply it.

That keeps `--write` on (because the wrapper is clean) AND tells Codex the source-files boundary (because Codex reads the staged file).

## Verify after dispatch

Once the rescue subagent returns the Codex job ID, open `~/.claude/plugins/data/codex-openai-codex/state/<workspace>/jobs/<job-id>.json`. The `request.write` field must be `true`. If it is `false`, the rescue subagent stripped `--write` and Codex will not be able to write to bd. Cancel the job (`node "$COMPANION" cancel <job-id>`, see references/cancellation.md if cancel fails) and re-dispatch with corrected wording. Catching this BEFORE the job runs costs no Codex tokens; catching it AFTER means the round's analysis must be salvaged manually as freeform text.
