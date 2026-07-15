---
name: codex
description: Delegate tasks to OpenAI Codex (GPT-5.5 default; GPT-5.6 sol/terra/luna available via --model) as background tasks for precision coding, code review, deliberation, and complex implementation. Always launch in background (run_in_background=true), continue working, then collect results with TaskOutput when needed.
allowed-tools: Bash, Read, Grep, Glob, TaskOutput, Edit, Write
---

# Codex

> Paths below use `{base}` as shorthand for this skill's base directory, provided automatically at the top of the prompt when the skill loads.

Codex is GPT-5.5 by default — with the GPT-5.6 family (`sol`/`terra`/`luna`) available via `--model` — a different model with a different reasoning manifold than Claude. It catches things you miss, thinks about problems differently, and arrives at solutions from a different angle. Use it as a genuine second brain, not just a subprocess. Its opinions, reviews, and implementations carry independent signal — when Codex disagrees with your approach, that disagreement is valuable.

Three modes of operation:

| Mode | Command | Sandbox | Web | Sessions | Purpose |
|------|---------|---------|-----|----------|---------|
| **think** | `think` | read-only | yes | ephemeral | Analysis, deliberation, research, review, second opinions |
| **review** | `review` | read-only | yes | ephemeral | Specialized code/PR diff review |
| **run** | `run` | full-access | yes | persisted | Implementation, coding, refactoring, bug fixes |

**Default to `think` for neutral plan review and high-risk workflow/config changes.**
Use adversarial framing only as a second-pass objection-mining step after a neutral verdict (see Review-Prompt Framing below).
**Choose think** when the user wants opinions, analysis, research, or a critique before implementation - no files will be modified.
**Choose review** when there is already a concrete diff, commit, or PR to inspect.
**Choose run** when the user wants code changes.
When unclear, default to **think** if the next useful step is judgment, and **run** only when the user clearly wants Codex to implement.

## Usage

```bash
# Think (read-only + web search)
{base}/scripts/codex.sh think "prompt" --dir /path/to/project
{base}/scripts/codex.sh think "prompt" --image screenshot.png --dir /project

# Run (full-access + web search)
{base}/scripts/codex.sh run "prompt" --dir /path/to/project
{base}/scripts/codex.sh run "prompt" --allowed-files "src/auth.py,src/auth_test.py" --dir /project
{base}/scripts/codex.sh run "prompt" --image mockup.png --dir /project
{base}/scripts/codex.sh run "prompt" --schema schema.json --dir /project
{base}/scripts/codex.sh run "prompt" --add-dir /other/path --dir /project

# Review (read-only, specialized)
{base}/scripts/codex.sh review --base main "Focus on security"
{base}/scripts/codex.sh review --commit abc123
{base}/scripts/codex.sh review --uncommitted

# Resume a previous run session
{base}/scripts/codex.sh resume --last "follow-up instruction"
{base}/scripts/codex.sh resume --session <SESSION_ID> "follow-up"
```

**Flags:** `--dir`, `--model`, `--effort`, `--sandbox`, `--image`, `--ephemeral`, `--schema`, `--add-dir`, `--allowed-files`

## Mode Decision Tree

1. Is there a plan, design, rule, hook, workflow, data contract, or migration proposal that needs challenge before it is acted on?
   - Use `think`.
2. Is there an existing diff/commit/PR and the task is to find defects in it?
   - Use `review`.
3. Is the task a bounded implementation with an explicit file allowlist?
   - Use `run`.
4. Is the task trivial, purely conversational, or requires user interaction mid-flight?
   - Do it yourself.

## Verified Adversarial-Review Catches

2026-05-13 benchmark evidence for CC-invoked Codex:

- `/codex think` caught 6/6 substantive plan-review issues in the sampled sessions.
- 3/3 initial spot-checked claims and 3/3 in-session plan-review claims were technically correct against canonical files or docs.
- 2/3 checked corrections were shipped verbatim to production code.
- `review` mode was underused in the sample: 0 invocations across 94 sessions; `think` had only 6 substantive invocations.
- The GitHub CI `Codex AI Review` bot is a different path. Phase 1/2 found 0/10 known post-merge defects caught, but Phase 3 corrected the parser and found 30 `Severity: BLOCKING` items across 14/30 sampled PRs plus 64 SUGGESTION items across 18/30 sampled PRs.

2026-05-14 manual sample of 10 CI `Severity: BLOCKING` items found real signal mixed with predictable false positives: roughly 7/10 were technically valid at review time, 3/10 were Python-version false positives, and one valid security finding was repeated across artifacts.

## CI Review Calibration

- Treat repeated same-root `BLOCKING` blocks as duplicates; fix the root cause once and do not inflate blocker counts.
- Verify Python language-semantics claims against the repo runtime before accepting them. `asyncio.TimeoutError is TimeoutError` and PEP 604 unions in `isinstance` are valid on modern Python.
- Require regression coverage for bug fixes, but only require `tests/test_regression.py` when the repo policy explicitly mandates that exact file.
- Track raw blocker count separately from independent high-value blocker rate.

Practical default: use `think` before changing high-risk CC behavior, hooks, rules, skills, plugin routing, memory indexes, MCP routing, merge/deploy policy, data contracts, billing/finance flows, agent tool grants, or data-classification tiers.

## Pre-Acceptance Verification

Codex suggestions usually carry useful independent signal, but do not blindly ship claims that depend on local CC runtime behavior. Before accepting:

1. Classify the claim:
   - General code/design claim: verify by reading the touched code and running targeted tests.
   - Local `~/.claude`/Claude Code behavior claim: verify against current files, docs, hooks, logs, or a live probe.
   - External/current-service claim: verify with the relevant current source.
2. If Codex names a file, line, hook event, setting, API shape, or CLI behavior, open the file or source and confirm it exists now.
3. If Codex proposes a hook or enforcement mechanism, prove the firing event and payload shape before building on it.
4. If verification is not possible in-session, downgrade the recommendation to a follow-up instead of baking it into production config.

Document important verified or rejected claims in memory when they are likely to affect future CC behavior.

## Review-Prompt Framing (2026-06-10 pilot; directional, n=12 packets)

Default review prompts to neutral assessment: "assess whether the reasoning/change supports its conclusion", not "find flaws" or "red-team." In the 2026-06-10 pilot, neutral framing reduced Claude clean-control FLAWED verdicts from 5/6 to 0/6 with one fewer catch; GPT still produced 3/6 clean-control failures and repeated platform-behavior counterclaims in both framings.

Treat verdicts as objection-generation until each decision-driving objection is quote/source-verified. For high-stakes reviews, run a neutral verdict first; use adversarial framing only as second-pass objection mining. Do not act on a single FLAWED verdict without evidence verification — same-input run-to-run verdict flips were observed.

## Failure Modes

- Codex cannot see local runtime/config drift unless you give it paths or let it read the current workspace.
- `think`/`review` run in a read-only sandbox that blocks network shell commands (`gh`, `curl`). If a review needs a diff/PR Codex can't fetch itself, feed the diff in-prompt (or via a file path) rather than expecting it to run `gh pr diff`.
- `think`/`review` now skip the user MCP stack by default (`--ignore-user-config`); they keep web search, `--model`, and `--effort` (all passed as CLI overrides). This is deliberate: loading the full configured MCP stack (graphite, stripe-sandbox, etc.) added minutes of startup and was observed to hang ~40min at MCP init on 2026-07-02, making an in-prompt review (which needs no MCP tools) unusable. If a review genuinely needs a project MCP tool, pass `--with-mcp` to load the stack. `run` mode is unchanged and still loads the full config/MCP. (Same root cause as the codex-runtime-hygiene rule: a read-only worker without `--ignore-user-config` still boots the whole MCP stack.)
- Codex can fabricate CC-specific internals such as hook payload fields, tool timing, or settings behavior; verify those against current files/docs/logs.
- `run` mode can overreach; always enforce an allowlist and inspect the diff.
- CI `Codex AI Review` comments are not equivalent to `/codex think` or `/codex review`; do not transfer benchmark conclusions between those paths without evidence.

## How to Invoke

1. **Always run in background.** Continue working or block on `TaskOutput`.
2. **Pass the user's intent as-is.** Don't over-engineer the prompt — Codex reads files and figures things out.
3. **Add context Codex can't see** — working directory, file paths, framework info, constraints from earlier in conversation.
4. **Always include a file allowlist** for `run` mode — see Scope Control below.
5. **Collect results** with `TaskOutput(task_id=..., block=True, timeout=300000)`.
6. **After run mode**, validate scope and review changes — see Scope Control below.

```python
Bash(command='{base}/scripts/codex.sh think "Is this auth design scalable?" --dir /project',
     run_in_background=True)
# → task_id

TaskOutput(task_id="...", block=True, timeout=300000)
# → SESSION: <uuid>\n---\n<response>
```

For complex tasks, add structure (see `references/prompt-engineering.md` for templates). For simple asks, just describe what you need.

## Scope Control (MANDATORY for run mode)

Codex will modify files outside its assigned scope. This is not a bug — it's how GPT-5.5 reasons about "completeness." You MUST constrain it.

### File Allowlist

Every `run` prompt MUST include an explicit file allowlist:

```
FILES YOU MAY MODIFY (and ONLY these files):
- path/to/file1.py
- path/to/file2.py

DO NOT modify any other files. DO NOT modify test expectations, test counts,
or assertion values to make tests pass — if tests fail, your changes are wrong.
DO NOT reformat, re-order, or normalize YAML/JSON files you read for context.
DO NOT move rules between files (e.g. promoting/demoting shared rules).
DO NOT run git commit, git add, git push, or any git write operations.
```

If you don't know the exact file list, derive it from the plan or use `think` first.

Also pass the allowlist to codex.sh for automated enforcement:
```bash
{base}/scripts/codex.sh run "prompt" --allowed-files "file1.py,file2.py" --dir /project
```

### Git Operations — CC Only

Codex `run` mode is for implementation ONLY. Codex must NEVER:
- `git commit` or `git add`
- `git push` or create PRs
- Run any git write operations

**Incident 2026-07-01 — `run` mode merged a PR despite an explicit DO-NOT-merge instruction.** A `run` task was told, in bold, "audit only; DO NOT merge; DO NOT git-write" and STILL performed the PR merge itself; its `--delete-branch` then removed the branch out from under its own worktree, corrupting its CWD (`getcwd` error) and exiting 1. The instruction alone is NOT a control. Therefore:
- Never give a `run` task PR-merge authority, or any task whose notion of "completeness" implies merging/pushing. Run PR audits/reviews in read-only `think`/`review` (or a Claude subagent); keep the merge step entirely on the CC side after collecting output.
- If a `run` task must operate inside a git worktree, expect that branch/worktree-mutating overreach can destroy its own CWD; prefer a stable `--dir` outside the mutated worktree and a tight `--allowed-files`.

After collecting Codex output, CC handles the entire git lifecycle:
1. Review all changes (`git diff --stat`, `git diff`)
2. Validate against file allowlist (restore unauthorized changes)
3. Re-run tests
4. Commit, push, PR via normal CC workflow

If Codex created commits during `run`, codex.sh auto-resets to the pre-run HEAD.
If that failed, manually reset: `git reset HEAD~N` then review the working tree.

### Post-Run Validation (MANDATORY)

After collecting Codex `run` output, ALWAYS:

1. Run `git status --short` in the `--dir` directory
2. Compare changed files against the allowlist from the prompt
3. For ANY file not in the allowlist: restore it immediately
4. Re-run tests AFTER restoring unauthorized changes
5. If Codex changed test expectations (counts, assertion values): treat as unauthorized — restore from origin

If `--allowed-files` was passed, codex.sh validates automatically and reports violations in stderr.
For manual validation:

```bash
# Pattern: validate and restore after every Codex run
ALLOWED="file1.py file2.py"  # from your prompt
cd <dir>
while IFS= read -r f; do
  if ! echo "$ALLOWED" | grep -qwF "$f"; then
    echo "UNAUTHORIZED: $f — restoring"
    git checkout -- "$f" 2>/dev/null || rm -f "$f"
  fi
done < <(git diff --name-only && git ls-files --others --exclude-standard)
```

## Error Handling

- **Empty output with error text**: API key issue or auth failure. Check env vars.
- **Empty output, no errors**: Codex timed out or crashed. Retry once; if it persists, reduce task scope.
- **Exit code in header**: Codex encountered issues but produced partial output. Review critically.
- **"session id: unknown"**: Codex CLI output format may have changed. Resume will not work for this session.
- **SCOPE VIOLATION in stderr**: codex.sh detected and auto-restored unauthorized file changes. Review the list to understand what Codex tried to do.

When Codex fails, do NOT silently drop the result. Report the failure to the user with the error details.

## Model Selection

The default model comes from `~/.codex/config.toml` — currently **`gpt-5.5`**. Default to it; only override with `--model` when there is a clear cost/performance reason.

| Task Type | Recommended | Flag |
|-----------|-------------|------|
| Typo fix, simple edit | gpt-5.1-codex-mini | `--model gpt-5.1-codex-mini --effort low` |
| Standard implementation | gpt-5.5 (default) | (no flag needed) |
| Security review, architecture | gpt-5.5 | `--effort xhigh` |
| Frontier reasoning / hardest tasks | gpt-5.6-sol (or -terra) | `--model gpt-5.6-sol --effort max` |
| Long multi-file refactor | gpt-5.1-codex-max | `--model gpt-5.1-codex-max` |

### GPT 5.6 family (opt-in via `--model`)

The GPT 5.6 models are available as `--model` overrides. They add two reasoning tiers beyond `xhigh`: **`max`** and **`ultra`**. The default stays `gpt-5.5` — reach for 5.6 only when a task needs frontier reasoning.

| Model | Character | Efforts | Account status |
|-------|-----------|---------|----------------|
| `gpt-5.6-terra` | Balanced, everyday frontier | low…xhigh, `max`, `ultra` | ✅ Verified callable on this ChatGPT-auth Codex account (2026-07-16). Solid everyday 5.6 pick. |
| `gpt-5.6-sol` | Most capable frontier model | low…xhigh, `max`, `ultra` | ✅ Verified callable (2026-07-16). Note: intermittently 400'd ("not supported with a ChatGPT account") earlier in July — account availability has fluctuated, so expect the occasional fallback. |
| `gpt-5.6-luna` | Fast & affordable | low…xhigh, `max` (no `ultra`) | ✅ Verified callable (2026-07-16). Fast tier; caps at `max` effort. |

- `ultra` effort requires `gpt-5.6-sol` or `gpt-5.6-terra`; `gpt-5.6-luna` caps at `max`.
- `codex.sh` forwards `--model`/`--effort` straight to `codex exec` with no allowlist, so any catalogued model + supported effort works with no further change.

## When Codex Shines

- **Second opinion** — its different training means it spots different bugs, suggests different patterns, and flags things you'd overlook
- **Neutral second opinion** — use `think` to assess whether the reasoning or change supports its conclusion. For high-stakes work, follow with an adversarial objection-mining pass and quote-verify each decision-driving objection
- **Parallel expertise** — while you work on feature A, Codex implements feature B or researches approach C
- **Deep reasoning tasks** — xhigh effort on complex algorithms, security analysis, architecture decisions

## When NOT to Use Codex

- Simple edits, typos, trivial changes — do them yourself
- Multi-file orchestration — you coordinate better across many files
- Conversational responses or explanations
- Tasks requiring mid-execution user interaction

## Parallelism

Launch multiple Codex tasks at once. Peek without blocking: `TaskOutput(task_id=..., block=False, timeout=0)`.

## AGENTS.md Sibling Convention

Each skill that is useful to Codex has an `AGENTS.md` file next to its `SKILL.md`. `AGENTS.md` contains the same procedure in Codex's expected format — same content, no CC-specific YAML.

When Codex needs a skill procedure (e.g. "how do I run the test sweep?"), look for `AGENTS.md` in the same directory as `SKILL.md`. If it exists, use it. If it doesn't exist, the skill is CC-only.

To find skills with `AGENTS.md`:
```bash
find ~/.claude/plugins/cache -name "AGENTS.md" | sort
```

Skills that have `codex_visible: true` in their `SKILL.md` frontmatter are guaranteed to have a sibling `AGENTS.md`.

## Self-Healing

If anything breaks, fix the skill files directly — you have authorization to edit anything under `{base}/`:
- `scripts/codex.sh` — wrapper script
- `SKILL.md` — this file
- `references/cli-reference.md` — CLI flags
- `references/prompt-engineering.md` — prompt templates
