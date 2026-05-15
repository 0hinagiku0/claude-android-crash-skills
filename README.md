# Claude Android Crash Skills

A set of [Claude Code](https://claude.com/claude-code) skills for Android crash triage and fix review, designed for teams using [Umeng U-APM](https://www.umeng.com/) or similar crash monitoring platforms.

**Three skills, two workflows**:

- **Workflow A** (semi-automated, original): `crash-analyzer` + `crash-fix-review` — human-in-loop, review after fix
- **Workflow B** (fully automated, v2): `crash-auto-loop` — Claude writes the fix, an independent Codex reviewer scores it across 9 dimensions, iterate up to 3 rounds, auto-commit + push when all green

## What's inside

| Skill | Trigger | Workflow | Purpose |
|---|---|---|---|
| **crash-analyzer** | `/analyze-crashes` | A (semi-auto) | Parse a crash CSV, auto-classify entries (A/B/C/D/E), generate a Markdown report, then interactively walk through fixes — branching to `auto_fix_bug`, applying minimum-change patches, running `detekt` + compile checks, committing on approval. |
| **crash-fix-review** | `/review-crash-fixes` | A (semi-auto) | An independent reviewer (run in a separate Claude session to avoid self-justification bias). Reviews fix commits on `auto_fix_bug` against 9 dimensions — root cause / new crash risk / minimum-change / verification / commit message / business impact / backward compat / privacy & compliance / error handling & observability. |
| **crash-auto-loop** ⭐ | `/auto-fix-crashes` | B (fully auto) | Automated end-to-end. Pulls crashes directly from Umeng admin via `chrome-devtools` MCP, classifies, generates fixes, invokes `codex` CLI as an independent reviewer scoring the 9 dimensions, loops until all-green or 3 rounds, then auto-commits and pushes to `auto_fix_bug`. Two AI models cross-checking each other. |

In **Workflow A**, `crash-analyzer` makes the fixes interactively, then a fresh Claude session runs `crash-fix-review` against the resulting commits. The human is the integrator.

In **Workflow B**, `crash-auto-loop` runs the whole pipeline in a single command, using two AI models (Claude as author, Codex as reviewer) to remove the self-review bias problem. The human only handles `B` class (business-decision) crashes and 3-round escalations.

## Design principles

Shared across all three skills:

1. **Minimum change first** — Recommended fix must satisfy quantitative thresholds (≤3 files / ≤30 lines / no new public API / no schema change / no thread model change). Refactoring options are flagged with ⚠️ and require explicit user choice.
2. **Real code, no hallucination** — Before suggesting any fix, the skill reads the actual stack-top file and greps for existing repair APIs in the codebase. If the project already has the standard fix for that crash type, the entry gets reclassified — surfacing the more valuable signal "the fix exists but isn't effective" rather than re-suggesting it.
3. **detekt-compatible** — Fixes must pass project `detekt` configuration before committing. Failed fixes auto-rollback via `git reset --hard HEAD`.
4. **Branch isolation** — All commits stay on `auto_fix_bug` branch. The skills never touch `master`/`dev`.

Specific to **crash-auto-loop (v2)**:

5. **Cross-model review** — Author (Claude) and reviewer (Codex) run on different model providers. The reviewer has no context about why the author made their choices — it only sees the code and the 9-dimension scoring rules. This kills self-justification bias structurally.
6. **9-dimension verdict contract** — Reviewer must output strict `[A-I] <emoji> <reason>` format. Skill parses with regex `^\[([A-I])\] (✅|⚠️|❌) (.+)$`. Any `❌` blocks commit; `⚠️` is allowed but logged in commit body as follow-up.
7. **3-round upper bound** — Author and reviewer iterate (fix code OR write rebuttal) up to 3 rounds. After 3, the crash is escalated with full round history. Prevents infinite hallucination loops.
8. **Three-fence push safety** — Before pushing, the skill verifies HEAD == `auto_fix_bug`, branch.remote == `origin`, and the push command itself contains no `--force` or `+refs`. Push failure escalates immediately, never retries.
9. **Never silent fallback** — If chrome-devtools fetch fails → DOM scrape; if DOM scrape fails → wait for manual CSV. Every fallback is recorded in the final report with cause.

## Installation

```bash
# Clone into a temp location, then copy each skill folder to the global Claude Code skills directory
git clone <this-repo-url> /tmp/skills
mkdir -p ~/.claude/skills
cp -r /tmp/skills/skills/crash-analyzer    ~/.claude/skills/
cp -r /tmp/skills/skills/crash-fix-review  ~/.claude/skills/
cp -r /tmp/skills/skills/crash-auto-loop   ~/.claude/skills/   # v2

# If you also use Codex CLI, mirror the same three folders to ~/.codex/skills/
mkdir -p ~/.codex/skills
cp -r ~/.claude/skills/crash-*   ~/.codex/skills/
```

## Prerequisites

### For Workflow A (`crash-analyzer` + `crash-fix-review`)

- Claude Code installed and authenticated
- Project root must contain `config/detekt/detekt.yml`
- An exported Umeng U-APM CSV to feed into `.cc_tmp/crash/inbox/`

### For Workflow B (`crash-auto-loop`) — additionally requires:

- **Codex CLI** ≥ 0.130 installed and authenticated: `brew install codex` or `npm i -g @openai/codex` (whichever your platform supports)
- **chrome-devtools MCP** installed with `--browserUrl` pointing to a real Chrome instance:
  ```bash
  claude mcp add --scope user chrome-devtools -- \
    npx -y chrome-devtools-mcp@latest --browserUrl http://127.0.0.1:9222
  ```
- **Dedicated Chrome profile with debug port** (Chrome 136+ silently ignores `--remote-debugging-port` on the default profile — security mitigation against cookie theft):
  ```bash
  open -na "Google Chrome" --args \
    --remote-debugging-port=9222 \
    '--remote-allow-origins=*' \
    --user-data-dir="$HOME/.cc_tmp/chrome-debug-profile" \
    --new-window \
    "https://apm.umeng.com/platform/<appkey>/error_analysis/crash"
  ```
  Log in to Umeng once in this isolated profile; credentials persist for future runs.

## Configuration

Edit `~/.claude/skills/crash-analyzer/config.yaml`:

- `home_packages`: your app's package prefixes (e.g. `com.yourcompany.app.`)
- `third_party_packages`: SDK / system layer prefixes that should be classified as "not your code"
- `known_fix_exceptions`: exception types with clear fixes (auto class A)
- `business_decision_exceptions`: exception types requiring business judgment (auto class B)
- `git_log_window_days`: how far back to scan git log for "already fixed" matches
- `auto_fix_branch`: branch name for fix commits (default: `auto_fix_bug`)

`crash-auto-loop` reads the same config (the classification decision tree is shared by reference).

For `crash-auto-loop`, also tune `recipes/umeng-recipe-example.yaml`:

- `project_appkey`: your Umeng app's 24-hex appkey (auto-recorded on first run)
- `project_name`: human-readable project label for reports
- `field_map`: JSON → CSV column mapping (jq syntax) if your Umeng tenant returns different field names

## Usage

### Workflow A — Semi-automated (original)

#### Step 1: Triage

```bash
# Put your Umeng U-APM CSV in:
.cc_tmp/crash/inbox/2026-05-13-crashes.csv

# Then in Claude Code:
/analyze-crashes
```

The skill will:

1. Check working tree is clean
2. Fetch `origin/master`, checkout/create `auto_fix_bug`, merge `origin/master`
3. Read `config/detekt/detekt.yml` to extract enabled rules
4. Parse the CSV (UTF-8 → GBK fallback)
5. Classify each entry (D → C → B → A → E)
6. Cross-reference with `git log --since=30.days` for "already fixed" matches
7. **Read stack-top file ±15 lines + grep for existing repair APIs** (most important step)
8. Generate `.cc_tmp/crash/reports/YYYY-MM-DD.md`
9. Ask whether to enter interactive fix mode

#### Step 2: Fix (interactive)

For each A/B/E entry:

- Skill presents 2-3 fix options (⭐ minimum change as default, ⚠️ refactor as option)
- You pick / skip / exit
- Skill applies the change
- Runs `./gradlew :<module>:detekt :<module>:compile<Variant>Kotlin`
- On failure: `git reset --hard HEAD && git clean -fd`, mark as failed, continue
- On success: show diff, ask commit/adjust/skip
- Commits to `auto_fix_bug` with structured message (never auto-pushes)

#### Step 3: Review (separate Claude session)

```
/review-crash-fixes
# or specify branch:
/review-crash-fixes some-other-branch
```

The reviewer:

- Reads each commit's full diff + surrounding code
- Greps for callers of changed functions/fields
- Walks 9 dimensions (A–I), assigning ✅ / ⚠️ / ❌
- Generates `.cc_tmp/crash/reviews/YYYY-MM-DD-<branch>.md`
- Computes overall verdict: 🟢 ship / 🟡 ship with notes / 🔴 BLOCK

#### Step 4: Apply review feedback

If `crash-fix-review` flagged BLOCK or improvement points, go back to `crash-analyzer` and request adjustments. The author skill doesn't see the review report directly — you summarize the feedback yourself, keeping the review pressure honest.

---

### Workflow B — Fully automated (v2)

```
/auto-fix-crashes
```

One command. The skill runs through all eight phases:

1. **Preflight** — Verify codex CLI, chrome-devtools MCP, detekt config
2. **Branch** — Switch to `auto_fix_bug`, merge `origin/master`, auto-bind upstream on first run
3. **Fetch** — Read `defaultVersionName` from `master:build.gradle`, compute last-7-days date window, invoke `mcp__chrome-devtools__evaluate_script` to run `fetch("/hsf/analysis/errorList", POST, ...)` inside the browser (reusing your login cookies)
4. **Classify** — Run the v1 decision tree (D → C → B → A → E)
5. **Sub-loop per A/B crash** (up to 3 rounds each):
   - Claude writes proposal + applies diff (uncommitted)
   - Pipe prompt + crash context + git diff to `codex exec --sandbox read-only`
   - Parse Codex's 9-dimension verdict table
   - Decide: all-green → commit; any ❌ → either fix code OR write rebuttal back to Codex; round 3 → escalate
6. **Build verify** — `./gradlew :module:compileReleaseKotlin/Javac` + `detekt`
7. **Commit + push** — Three-fence safety, then `git push origin auto_fix_bug`
8. **Summary report** — `.cc_tmp/crash/reports/auto-<date>.md` with success / failure / escalate / Codex call stats

Human intervention only happens for:

- B-class crashes requiring business judgment (skill skips them, lists in report)
- 3-round escalations (skill writes full round history to `.cc_tmp/crash/escalate/<id>.md`)
- Push failures (skill never retries, surfaces stderr)

See `skills/crash-auto-loop/docs/e2e-verification.md` for the 4-item end-to-end acceptance checklist.

## What this skill set is NOT

- Not a replacement for `git revert` / `git bisect` — those are still the right tools for "production rollback"
- Not a way to bypass code review — `crash-fix-review` and Codex are first passes, not substitutes for team review
- Not for non-Android projects — the conventions (detekt, R8 mapping, proguard, AndroidManifest) are Android-specific
- Not for projects without a CSV-based crash export (Workflow A) or without web-backed Umeng/Bugly/Crashlytics (Workflow B) — adapting to other crash platforms requires rewriting `parsing.md` and the recipe yaml
- **`crash-auto-loop` is not free** — Codex CLI calls are paid; budget accordingly. Each crash uses ~1-3 Codex review calls × ~5K tokens

## Caveats

### Shared

- **R8/proguard mapping**: when crash CSV stack lines show obfuscated identifiers (e.g. `r8-map-id-xxx`), the skill cannot resolve them. Feed it `mapping.txt` separately, or accept that the report will fall back to "best guess" for those entries.
- **detekt baseline drift**: if your project hasn't kept `detekt-baseline.xml` clean, the skill may report failures unrelated to its changes.
- **`auto_fix_bug` branch tracking trap**: when first created via `git checkout -b auto_fix_bug origin/master`, git's default behavior sets tracking to `origin/master`. This caused a real incident — see "Lessons learned". `crash-auto-loop` mitigates this by force-binding the branch to `origin/auto_fix_bug` via `git push -u` on first run.

### Specific to `crash-auto-loop`

- **Browser session expires** — Umeng's login cookies expire periodically; you'll need to re-login in the isolated debug profile occasionally
- **Umeng API drift** — the skill hardcodes the request body schema for `/hsf/analysis/errorList`. If Umeng changes their internal API, the recipe yaml needs updating
- **Codex review format drift** — the skill expects strict `[A-I] <emoji> <reason>` output. If Codex prompts evolve, may need to retune `instructions/review-prompt.md`
- **Single project only** — recipe stores one `project_appkey`. If you maintain multiple apps, you need separate recipe files

## Lessons learned (from real production use)

This skill set was developed and tested on a production Android codebase. Some incidents and surprises worth noting:

1. **The "E class skip" trap** — early `crash-analyzer` let users skip "unclassified" entries too easily. Now §5.5 enforces a reality check (grep the actual code) before any A/B/E entry can be marked as "not fixable here."
2. **The "auto-commit without review" trap** — early `crash-analyzer` committed fixes immediately, then reviewed *after*. If review failed, the failed fix polluted git history. `crash-auto-loop` reversed this: review first, commit only on all-green.
3. **The Android Studio Push dialog trap** — `git checkout -b auto_fix_bug origin/master` auto-sets tracking to `origin/master`. Android Studio's Push dialog defaults Target to the tracking branch. One day, a user pressed Push without changing Target, and `auto_fix_bug` commits were pushed directly onto `origin/master`. **`crash-auto-loop` mitigates this by `git push -u origin auto_fix_bug` on first run, binding the branch correctly.**
4. **Chrome 136+ debug port silent rejection** (`crash-auto-loop` discovery) — `--remote-debugging-port=9222` is silently ignored when used with the default user-data-dir. Process arguments show the flag is set, but the port never actually listens. Fix: always use a dedicated `--user-data-dir` for the debug profile.
5. **`codex review --uncommitted` ⊥ `[PROMPT]`** (`crash-auto-loop` discovery) — Codex CLI v0.130 rejects custom review prompts when `--uncommitted` is used. Workaround: use `codex exec --sandbox read-only - < input.md` with the prompt + git diff combined into a single stdin stream.
6. **Cross-AI verdict contract drift** — In early prototypes Codex sometimes output `[OK]/[WARN]/[FAIL]` instead of `✅/⚠️/❌`. Explicit "must use these exact emoji, do not substitute" in the prompt fixed it. The output contract section in `instructions/review-prompt.md` is intentionally strict and verbose for this reason.
7. **The `--remote-allow-origins=*` zsh glob trap** — zsh expands unquoted `*`. The Chrome launch command must use quoted `'*'` or the flag silently becomes empty, and WebSocket connections to port 9222 get 403'd. Easy 30-minute debug if you don't know.

## License

MIT — see [LICENSE](LICENSE).

## Contributing

This is a personal release. Issues and PRs welcome, but no SLA on response.

If you adapt this for a different crash platform (Firebase, Bugsnag, Sentry), please open a PR — the parsing layer is the only platform-specific part.

If you want to port `crash-auto-loop` to use a different LLM as the reviewer (e.g. Gemini, GLM, DeepSeek), the abstraction is fairly clean — `codex exec - < input.md > output.md` is just a subprocess pipe, and the 9-dimension contract is in `instructions/review-prompt.md`.
