# Claude Android Crash Skills

A pair of [Claude Code](https://claude.com/claude-code) skills for Android crash triage and fix review, designed for teams using [Umeng U-APM](https://www.umeng.com/) or similar crash monitoring platforms.

## What's inside

| Skill | Trigger | Purpose |
|---|---|---|
| **crash-analyzer** | `/analyze-crashes` | Parse a crash CSV, auto-classify entries (A/B/C/D/E), generate a Markdown report, then interactively walk through fixes — branching to `auto_fix_bug`, applying minimum-change patches, running `detekt` + compile checks, committing on approval. |
| **crash-fix-review** | `/review-crash-fixes` | An independent reviewer (run in a separate Claude session to avoid self-justification bias). Reviews fix commits on `auto_fix_bug` against 9 dimensions — root cause / new crash risk / minimum-change / verification / commit message / business impact / backward compat / privacy & compliance / error handling & observability. |

The two are designed to work together: `crash-analyzer` makes the fixes, then a fresh Claude session runs `crash-fix-review` against the resulting commits.

## Design principles

1. **Minimum change first** — Recommended fix must satisfy quantitative thresholds (≤3 files / ≤30 lines / no new public API / no schema change / no thread model change). Refactoring options are flagged with ⚠️ and require explicit user choice.
2. **Real code, no hallucination** — Before suggesting any fix, the skill reads the actual stack-top file and greps for existing repair APIs in the codebase. If the project already has the standard fix for that crash type, the entry gets reclassified — surfacing the more valuable signal "the fix exists but isn't effective" rather than re-suggesting it.
3. **detekt-compatible** — Fixes must pass project `detekt` configuration before committing. Failed fixes auto-rollback via `git reset --hard HEAD && git clean -fd`.
4. **Never auto-push** — All commits stay on `auto_fix_bug` branch. The skill never pushes to remote, never touches `master`/`dev`.
5. **Independent reviewer** — `crash-fix-review` is designed to run in a separate Claude session so the reviewer has no memory of why the fix author made their choices. This makes critique honest.

## Installation

```bash
# Clone into the global Claude Code skills directory
git clone https://github.com/<your-username>/claude-android-crash-skills.git /tmp/skills
mkdir -p ~/.claude/skills
cp -r /tmp/skills/skills/crash-analyzer ~/.claude/skills/
cp -r /tmp/skills/skills/crash-fix-review ~/.claude/skills/
```

If you also use [Codex CLI](https://github.com/openai/codex-cli), mirror to `~/.codex/skills/` as well.

## Configuration

Edit `~/.claude/skills/crash-analyzer/config.yaml`:

- `home_packages`: your app's package prefixes (e.g. `com.yourcompany.app.`)
- `third_party_packages`: SDK / system layer prefixes that should be classified as "not your code"
- `known_fix_exceptions`: exception types with clear fixes (auto class A)
- `business_decision_exceptions`: exception types requiring business judgment (auto class B)
- `git_log_window_days`: how far back to scan git log for "already fixed" matches
- `auto_fix_branch`: branch name for fix commits (default: `auto_fix_bug`)

## Usage

### Step 1 — Triage

```
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

### Step 2 — Fix (interactive)

For each A/B/E entry:

- Skill presents 2-3 fix options (⭐ minimum change as default, ⚠️ refactor as option)
- You pick / skip / exit
- Skill applies the change
- Runs `./gradlew :<module>:detekt :<module>:compile<Variant>Kotlin`
- On failure: `git reset --hard HEAD && git clean -fd`, mark as failed, continue
- On success: show diff, ask commit/adjust/skip
- Commits to `auto_fix_bug` with structured message

### Step 3 — Review (separate Claude session)

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

### Step 4 — Apply review feedback

If `crash-fix-review` flagged BLOCK or improvement points, go back to `crash-analyzer` and request adjustments. The author skill doesn't see the review report directly — you summarize the feedback yourself, keeping the review pressure honest.

## What this skill set is NOT

- Not a replacement for `git revert` / `git bisect` — those are still the right tools for "production rollback"
- Not a way to bypass code review — `crash-fix-review` is a first pass, not a substitute for team review
- Not for non-Android projects — the conventions (detekt, R8 mapping, proguard, AndroidManifest) are Android-specific
- Not for projects without a CSV-based crash export — currently parses Umeng U-APM CSV format; adapting to Firebase Crashlytics / Bugsnag would require rewriting `parsing.md`

## Caveats

- **R8/proguard mapping**: when crash CSV stack lines show obfuscated identifiers (e.g. `r8-map-id-xxx`), the skill cannot resolve them. You need to feed it `mapping.txt` separately, or accept that the report will fall back to "best guess" for those entries.
- **detekt baseline drift**: if your project hasn't kept `detekt-baseline.xml` clean, the skill may report failures unrelated to its changes. Use `--no-verify` only if you've verified the failure is pre-existing.
- **`auto_fix_bug` branch tracking trap**: when the skill creates `auto_fix_bug` via `git checkout -b auto_fix_bug origin/master`, git's default behavior sets tracking to `origin/master`. This caused a real incident — see "Lessons learned" below.

## Lessons learned (from real production use)

This skill was used on a production Android codebase and triggered three workflow incidents worth noting:

1. **The "E class skip" trap** — early versions let the user skip "unclassified" entries too easily. Now §5.5 enforces a reality check (grep the actual code) before any A/B/E entry can be marked as "not fixable here."
2. **The "auto-commit without review" trap** — early versions committed fixes immediately, then reviewed *after*. If review failed, the failed fix polluted git history. The intended workflow is staged → review → commit, but the skill's SKILL.md still reflects the old commit-first flow. Planned fix in `_design-docs/`.
3. **The Android Studio Push dialog trap** — `git checkout -b auto_fix_bug origin/master` auto-sets tracking to `origin/master`. Android Studio's Push dialog defaults Target to the tracking branch. One day, a user pressed Push without changing Target, and `auto_fix_bug` was pushed directly onto `origin/master`. **Use `git checkout --no-track -b auto_fix_bug origin/master` to prevent this.** (Not yet patched in this release.)

## License

MIT — see [LICENSE](LICENSE).

## Contributing

This is a personal release. Issues and PRs welcome, but no SLA on response.

If you adapt this for a different crash platform (Firebase, Bugsnag, Sentry), please open a PR — the parsing layer is the only platform-specific part.
