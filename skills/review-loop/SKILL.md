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

`/review-loop <target-file> [--threshold N] [--max-rounds N] [--cursor-model ID] [--codex-model ID]
[--codex-effort LEVEL]`

- `target-file` (required): the document under review, repo-relative (e.g. `docs/plans/foo.md`).
- `--threshold`: assurance % both reviewers must reach with a GO verdict. Default **95**.
- `--max-rounds`: hard cap. Default **8**. Hitting it without convergence = report and stop; never
  loop forever on a disagreement.
- `--cursor-model`: model for the Cursor reviewer. Default **`grok-4.7-high`**.
  Any id from `agent --list-models` works (e.g. `grok-4.7-xhigh`, `cursor-grok-4.6-high`,
  `claude-opus-5-thinking-high`); parameterized overrides too. Cursor bakes reasoning effort into
  the model id (`-low` / `-medium` / `-high` / `-xhigh`), so pick the tier in the id itself.
  Note the naming break: Grok 4.7 ids have **no `cursor-` prefix** (`grok-4.7-high`), while
  4.5/4.6 ids do (`cursor-grok-4.6-high`) — copy the id exactly as `agent --list-models` prints it.
- `--codex-model`: model for the Codex reviewer, passed as `codex exec -m <id>`. Default
  **`gpt-6-astra`**. Current lineup, deepest first:
  - **`gpt-6-astra`** — OpenAI's most capable model for complex, demanding work. The default; use
    it for real review work.
  - **`gpt-5.6-sol`** — reliable agentic workhorse. Good when Astra is rate-limited or a round is
    cheap; it was the previous default and still reviews well.
  - **`gpt-5.6-terra`** — balanced everyday model.
  - **`gpt-5.6-luna`** — fast and affordable. Fine for a quick re-verification round, weak as a
    primary reviewer.

  The account's available slugs are listed in `~/.codex/models_cache.json` (`.models[].slug`) —
  read it rather than guessing. If the configured model isn't in that list (older CLI, different
  plan), drop `-m` entirely and let the account default apply; say so in the round report.
- `--codex-effort`: reasoning depth for the Codex reviewer, passed as
  `-c model_reasoning_effort=<level>`. Default **`xhigh`**. Levels: `low`, `medium`, `high`,
  `xhigh`, `max`, `ultra`. Review is the deep-reasoning case — do not drop below `high` without
  the user asking. (Left unset, Codex picks the model's own default — `medium` for Astra, `low`
  for the 5.6 family — both far too shallow here.)
- **Heterogeneity rule:** the two reviewers should be different model families (the defaults
  satisfy this: Grok 4.7 from xAI vs GPT-6 Astra from OpenAI). If a user override makes both reviewers
  the same family, point out the lost diversity once, then proceed with their choice.
- **Model ids drift.** These defaults are point-in-time. If a CLI rejects one as unknown, list what
  the account actually has (`agent --list-models`, `~/.codex/models_cache.json`), pick the nearest
  equivalent tier, and tell the user what you substituted — never silently fall back to a weak model.

## Preflight (fail fast, tell the user exactly what's missing)

1. `export PATH="$HOME/.local/bin:$PATH"` in every Bash call — both CLIs typically live there.
   `jq` must also be on PATH (the Cursor reviewer's live stream is parsed with it); if missing,
   tell the user to install it (`brew install jq` / their package manager).
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
# -m: the --codex-model value if given, else the gpt-6-astra default.
# -c model_reasoning_effort: the --codex-effort value if given, else xhigh. Codex's own default is
# "medium" (Astra) or "low" (5.6), so this flag is not optional — without it the reviewer skims.
# < /dev/null is required: run non-interactively, `codex exec` otherwise sits waiting on stdin
# ("Reading additional input from stdin...") and the background task never finishes.
codex exec -m "gpt-6-astra" -c model_reasoning_effort="xhigh" \
  "$(cat <scratchpad>/round-N-prompt.md)" < /dev/null > <scratchpad>/round-N-codex.out 2>&1
```

`codex exec` must also be run from inside the git repo — outside one it exits immediately with
"Not inside a trusted directory and --skip-git-repo-check was not specified."
```bash
export PATH="$HOME/.local/bin:$PATH"
set -o pipefail
# Stream reasoning live (like Codex): stream-json + --stream-partial-output emits thinking/answer
# deltas as they happen; jq --unbuffered turns them into plain text that grows in the file in real
# time. Plain `--output-format text` buffers and only writes once the run completes — that's why
# the Cursor file used to appear all-at-once. Reviewer stderr goes to a separate file so it never
# corrupts the JSON stream; if the .out is empty, read the .err for the failure.
agent -p "$(cat <scratchpad>/round-N-prompt.md)" --model "grok-4.7-high" \
  --output-format stream-json --stream-partial-output --trust \
  2> <scratchpad>/round-N-cursor.err \
| jq --unbuffered -rj '
    if   .type=="thinking"  and .subtype=="delta"     then .text
    elif .type=="thinking"  and .subtype=="completed" then "\n\n--- verdict ---\n"
    elif .type=="assistant" and (.timestamp_ms!=null) then (.message.content[]? | select(.type=="text") | .text)
    else empty end
  ' > <scratchpad>/round-N-cursor.out
# --model: the --cursor-model value if given, else the grok-4.7-high default.
# The .out streams reasoning, a "--- verdict ---" separator, then the final answer (VERDICT footer
# lands at the tail, so the exit-check grep is unaffected).
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
