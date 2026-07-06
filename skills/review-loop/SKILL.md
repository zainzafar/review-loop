---
name: review-loop
description: Run an adversarial review loop on a plan/spec/document using two external CLI reviewers (OpenAI Codex + Cursor Agent) in parallel, adjudicating their findings against the actual codebase and patching the document each round, until BOTH return GO at or above an assurance threshold (default 95%). Use when the user asks to "adversarially review X", "get codex/cursor to review", "review until GO", "run the review loop", or wants multi-model sign-off before implementation.
---

# Adversarial review loop (Codex + Cursor, iterate-until-GO)

Reviews a document with two *heterogeneous* external models until both certify it. The value over
spawning more Claude subagents is model diversity — different models have different blind spots.
Claude (you) is the **adjudicator**, never a rubber stamp: every finding gets verified against the
actual repo before being conceded, and the document is patched between rounds.

## Arguments

`/review-loop <target-file> [--threshold N] [--max-rounds N] [--cursor-model ID] [--codex-model ID]`

- `target-file` (required): the document under review, repo-relative (e.g. `docs/plans/foo.md`).
- `--threshold`: assurance % both reviewers must reach with a GO verdict. Default **95**.
- `--max-rounds`: hard cap. Default **8**. Hitting it without convergence = report and stop; never
  loop forever on a disagreement.
- `--cursor-model`: model for the Cursor reviewer. Default **`claude-opus-4-8-thinking-high`**.
  Any id from `agent --list-models` works; parameterized overrides too
  (e.g. `'claude-opus-4-8[context=1m,effort=high,fast=false]'`).
- `--codex-model`: model for the Codex reviewer, passed as `codex exec -m <id>`. Default: omit
  the flag and let the account default apply (Codex runs OpenAI models only).
- **Heterogeneity rule:** the two reviewers should be different model families (the defaults
  satisfy this: Opus 4.8 vs GPT-5.x-codex). If a user override makes both reviewers the same
  family, point out the lost diversity once, then proceed with their choice.

## Preflight (fail fast, tell the user exactly what's missing)

1. `export PATH="$HOME/.local/bin:$PATH"` in every Bash call — both CLIs typically live there.
2. `codex login status` must say logged in; `agent status` (or `cursor-agent status`) must say
   logged in. If either fails, stop and give the user the login command (`codex` first run /
   `agent login`).
3. Target file exists and is saved. Identify the repo's **binding convention files** — CLAUDE.md
   at the repo root plus any AGENTS.md files in the subtrees the document touches — and name them
   in every prompt.
4. Both CLIs are invoked **read-only**: Codex via plain `codex exec` (default sandbox — no
   `--full-auto`, no sandbox overrides), Cursor via `agent -p ... --trust` and **NEVER**
   `--force` / `-f` / `--yolo` (those enable file modification). Reviewers read and report;
   only Claude patches files.

## The loop (one round)

### 1. Compose the round prompt

Write it to the session scratchpad (`round-N-prompt.md`). Rounds escalate — re-running the same
attack class finds nothing new. Ladder (adapt to the document type; calibrated from a 6-round
production run):

- **R1 — fact-check:** verify every code reference/claim in the document against the repo
  (file:line evidence required); schema/convention compliance; concurrency traces; gaps an
  implementer would have to invent.
- **R2 — fix-verification + new-material attack:** judge each prior fix FIXED/PARTIAL/WRONG;
  attack only what changed (revisions inject fresh bugs).
- **R3 — method change:** end-to-end journey simulation, authorization matrix per operation,
  failure modes (infra down, partial deploy, races with ops actions), performance/N+1.
- **R4 — final gate:** implementation dry-run (write the actual schema/API contracts/state
  machines on paper), ops-runbook dry-run, pre-mortem with "is each failure mode detectable by
  the current analytics" check. Findings must be blocker-class or labeled polish; explicit
  permission to return a clean verdict.
- **R5 — spec integrity:** attack the DOCUMENT, not the design: dangling "as vN" references,
  internal contradictions between patched sections, decision→test→signal traceability matrix,
  "is it buildable by someone who wasn't in the review rounds."
- **R6+ — GO/NO-GO:** verify last round's fixes landed + one free-choice sweep + structured verdict.

Every prompt ends with this exact footer requirement:

```
Then give your verdict in EXACTLY this format on the last lines of your reply:

VERDICT: GO or NO-GO
ASSURANCE: <integer 0-100>%
BLOCKING ITEMS: <none | numbered list>

ASSURANCE = your confidence that a competent engineer implementing this document as written
ships correct, secure, convention-compliant work without needing a decision the document failed
to make. Calibrate honestly — do not inflate to end the process, do not deflate to seem
rigorous. Every point below the threshold must be attributable to a specific listed blocking
item; vague unease does not lower the number. An earned GO is as valuable as a finding —
manufacturing findings to appear thorough is a review failure.
```

Prompts also always include: the target file path, the repo's binding convention files,
"findings need file:line or traced-interleaving evidence, unevidenced findings don't count",
and — from round 2 on — "do not re-open items adjudicated in the document's disposition
appendices."

### 2. Run both reviewers in parallel, uncontaminated

Neither reviewer ever sees the other's output — convergence between independent reviews is the
strongest signal this process produces. Launch both as background Bash tasks from the repo root:

```bash
export PATH="$HOME/.local/bin:$PATH"
# add: -m "<codex-model>" only if --codex-model was given
codex exec "$(cat <scratchpad>/round-N-prompt.md)" > <scratchpad>/round-N-codex.out 2>&1
```
```bash
export PATH="$HOME/.local/bin:$PATH"
agent -p "$(cat <scratchpad>/round-N-prompt.md)" --model "claude-opus-4-8-thinking-high" --output-format text --trust > <scratchpad>/round-N-cursor.out 2>&1
# --model: the --cursor-model value if given, else the claude-opus-4-8-thinking-high default
```

Expect 3–15 minutes each; use `run_in_background: true` and generous timeouts. If one CLI errors
(auth expiry, rate limit), report it and continue with a degraded single-reviewer round only if
the user confirms — the exit condition requires both.

### 3. Adjudicate (the step that makes this work)

- **Verify before conceding.** Check every finding against the actual code (grep/read the cited
  file:line). Reviewers cite confidently and are sometimes wrong.
- **Fact convergence beats severity opinion.** If both reviewers flag the same underlying fact —
  even grading it differently (one "blocker", one "polish") — fix the fact.
- **Rebut with evidence only.** A rebuttal needs a repo citation or a traced interleaving, never
  "seems fine". Record rebuttals in the disposition appendix too.
- **Expect the severity ladder to descend** across rounds (design → mechanism → contract →
  document). If round N+1 finds *worse* problems than round N, your patches are injecting bugs —
  slow down.

### 4. Patch + paper trail

Apply accepted fixes to the target document. Bump its version marker (v2, v3, …). Append a
disposition appendix per round: conceded-and-fixed / rebutted-with-evidence / accepted-as-risk.
The appendices are load-bearing: they're what lets later rounds say "don't re-open adjudicated
items" and what makes the final document auditable.

### 5. Exit check

Parse both `VERDICT:` / `ASSURANCE:` footers (grep the output tails; if a reviewer omitted the
footer, one targeted re-ask for just the verdict block is allowed).

- **Both GO and both ASSURANCE ≥ threshold → done.** Record final verdicts in the document's
  appendix, report to the user, and ask whether to commit (never commit without being asked).
- One NO-GO → fix its numbered blocking items (verified first), then next round is a **targeted
  re-verification** for that reviewer only ("your N items were fixed thus: …; verify and re-issue
  the verdict; do not re-open adjudicated items"). A reviewer that already passed is not re-run
  unless the fixes touched something it certified.
- Max rounds hit → stop, summarize the unresolved disagreement, hand the decision to the user.

## Report format (each round, to the user)

Lead with the scoreboard (`Codex: NO-GO @ 90%, 2 items · Cursor: GO @ 96%`), then per finding:
conceded (with the fix) or rebutted (with the evidence). Keep the full reviewer outputs in the
scratchpad, not the chat.

## Cost & duration expectations

Each round ≈ one Codex + one Cursor agent run over a large document + repo context (several
minutes and meaningful token spend on the user's ChatGPT/Cursor subscriptions). A document that
starts solid converges in 2–3 rounds; a fresh complex plan can take 5–7. Tell the user the round
number and scoreboard each iteration so they can stop the loop at any time.
