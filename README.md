# review-loop

An adversarial **review-until-GO** loop for [Claude Code](https://claude.com/claude-code).

It reviews a plan, spec, or design document with **two heterogeneous external CLI reviewers** —
[OpenAI Codex](https://github.com/openai/codex) and [Cursor Agent](https://cursor.com/cli) —
running in parallel, then iterates round after round until **both** return a `GO` verdict at or
above an assurance threshold (default 95%).

The value over spawning more Claude subagents is **model diversity**: different model families have
different blind spots, and convergence between two independent reviews is a strong signal. Claude
stays the **adjudicator**, never a rubber stamp — every finding is verified against the actual repo
before it's conceded, and the document is patched between rounds with an auditable disposition
trail.

## Requirements

You supply and pay for the two external reviewer CLIs (they bill against your own subscriptions):

- **[OpenAI Codex CLI](https://github.com/openai/codex)** — the `codex` binary on your `PATH`, and
  logged in (`codex` on first run walks you through login). Runs OpenAI models; uses your
  ChatGPT/Codex subscription.
- **[Cursor Agent CLI](https://cursor.com/cli)** — the `agent` binary on your `PATH`, and logged in
  (`agent login`). Uses your Cursor subscription.
- **[`jq`](https://jqlang.github.io/jq/)** on your `PATH` — used to render the Cursor reviewer's
  live reasoning stream. `brew install jq` (or your package manager).

Notes:

- The skill prepends `export PATH="$HOME/.local/bin:$PATH"` to its shell calls, since both CLIs
  commonly install there. If yours live elsewhere, make sure they're on your `PATH`.
- The default Cursor reviewer model (`cursor-grok-4.5-high`) is a point-in-time example.
  Override it with `--cursor-model` / `--codex-model` (see Usage). Run `agent --list-models` to see
  what's available to you.
- Both reviewers are always invoked **read-only** — they read and report; only Claude patches files.

## Install (plugin — recommended)

In Claude Code:

```
/plugin marketplace add zainzafar/review-loop
/plugin install review-loop@zainzafar
```

You get one-command install and updates. Invoke it as `/review-loop:review-loop <file>` (the
namespaced command), or just describe the task ("adversarially review this plan until GO").

## Install (manual fallback)

```
git clone https://github.com/zainzafar/review-loop.git
cp -r review-loop/skills/review-loop ~/.claude/skills/review-loop
```

Then invoke it as `/review-loop <file>`.

## Usage

```
/review-loop <target-file> [--threshold N] [--max-rounds N] [--cursor-model ID] [--codex-model ID]
```

- `target-file` (required) — the document under review, repo-relative (e.g. `docs/plans/foo.md`).
- `--threshold` — assurance % both reviewers must reach with a GO verdict. Default `95`.
- `--max-rounds` — hard cap; hitting it without convergence reports and stops. Default `8`.
- `--cursor-model` — model for the Cursor reviewer. Default `cursor-grok-4.5-high`.
- `--codex-model` — model for the Codex reviewer (`codex exec -m <id>`). Default: account default.

Example:

```
/review-loop docs/plans/payments-refactor.md --threshold 97 --max-rounds 6
```

> Each round runs one Codex and one Cursor review over your document plus repo context — several
> minutes and real subscription credits per round. A document that starts solid converges in 2–3
> rounds; a fresh complex plan can take 5–7. The scoreboard is reported every round so you can stop
> at any time.

## How it works

Each round:

1. **Compose an escalating prompt** — rounds change attack class (fact-check → fix-verification →
   journey/authz/failure-mode simulation → implementation & ops dry-run → spec integrity →
   GO/NO-GO), so re-runs keep finding new things instead of repeating.
2. **Run both reviewers in parallel, uncontaminated** — neither sees the other's output.
3. **Adjudicate** — Claude verifies every finding against the actual code before conceding; fact
   convergence beats severity opinion; rebuttals require repo evidence.
4. **Patch + paper trail** — apply accepted fixes, bump the document version, append a per-round
   disposition appendix (conceded-and-fixed / rebutted-with-evidence / accepted-as-risk).
5. **Exit check** — both `GO` and both `≥ threshold` → done (asks before committing). One `NO-GO` →
   fix and targeted re-verify. Max rounds → stop and hand the decision back to you.

See [`skills/review-loop/SKILL.md`](skills/review-loop/SKILL.md) for the full instructions.

## License

[MIT](LICENSE) © Zain Zafar
