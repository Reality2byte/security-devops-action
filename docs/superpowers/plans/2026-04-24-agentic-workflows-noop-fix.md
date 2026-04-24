# Agentic Workflows noop Fix — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stop the three gh-aw agentic workflows from filing false-positive `[aw] ... failed` issues that page the on-call IcM rotation.

**Architecture:** For all three agentic workflows, enable the `noop` safe output with `report-as-issue: false` (lets the agent explicitly signal "nothing to do" without itself filing an issue) and set `safe-outputs.report-failure-as-issue: false` (blocks the auto-filed failure issue even when no output is produced). Update the `msdo-issue-assistant` prompt so its "don't respond" rules now direct the agent to call the `noop` tool explicitly. Regenerate the three `.lock.yml` files with `gh aw compile` and ship everything in one PR.

> **Note on syntax:** gh-aw v0.61.0 rejects `noop: true` as a boolean. The correct YAML shape is an object: `noop:\n    report-as-issue: false`. All YAML blocks below use that shape. If you see `noop: true` anywhere, the compile will fail with "value must be false. Expected format: {...}".

**Tech Stack:** GitHub Actions, gh-aw CLI v0.61.0, YAML, Markdown prompts.

**Spec:** [docs/superpowers/specs/2026-04-24-agentic-workflows-noop-fix-design.md](../specs/2026-04-24-agentic-workflows-noop-fix-design.md)

**Branch:** `fix/agentic-workflows-noop` (already created; the spec is committed there as `e6b5cfa`)

---

## Task 1: Verify gh-aw CLI is installed at the right version

**Files:** none (local tooling check)

- [ ] **Step 1: Check the gh-aw version**

Run:
```bash
gh aw version
```

Expected output contains `v0.61.0` (this matches the version recorded in the existing lock-file headers at [.github/workflows/msdo-issue-assistant.lock.yml:15](../../.github/workflows/msdo-issue-assistant.lock.yml#L15)).

If `gh aw` is not installed, install it first:
```bash
gh extension install github/gh-aw
```

If a different version is installed, upgrade:
```bash
gh extension upgrade gh-aw
```

- [ ] **Step 2: Confirm we are on the right branch**

Run:
```bash
git branch --show-current
```

Expected: `fix/agentic-workflows-noop`

If not on that branch:
```bash
git checkout fix/agentic-workflows-noop
```

---

## Task 2: Edit `ci-doctor.md` safe-outputs

**Files:**
- Modify: [.github/workflows/ci-doctor.md](../../.github/workflows/ci-doctor.md) (lines 32-40)

- [ ] **Step 1: Replace the `safe-outputs` block**

In [.github/workflows/ci-doctor.md](../../.github/workflows/ci-doctor.md), replace this exact block:

```yaml
safe-outputs:
  noop: false
  create-issue:
    max: 1
  add-labels:
    allowed: [ci-failure, flaky-test, build-failure, dependency-issue, needs-maintainer]
  add-comment: null
  create-pull-request: null
```

With:

```yaml
safe-outputs:
  noop:
    report-as-issue: false
  report-failure-as-issue: false
  create-issue:
    max: 1
  add-labels:
    allowed: [ci-failure, flaky-test, build-failure, dependency-issue, needs-maintainer]
  add-comment: null
  create-pull-request: null
```

Changes: `noop: false` replaced with the `noop:\n    report-as-issue: false` object form (enables the noop tool without having it file its own issue), plus `report-failure-as-issue: false` inserted as the next key.

- [ ] **Step 2: Verify the edit**

Run:
```bash
grep -nE "noop|report-failure-as-issue|report-as-issue" .github/workflows/ci-doctor.md
```

Expected output:
```
33:  noop:
34:    report-as-issue: false
35:  report-failure-as-issue: false
```

---

## Task 3: Edit `msdo-breach-monitor.md` safe-outputs

**Files:**
- Modify: [.github/workflows/msdo-breach-monitor.md](../../.github/workflows/msdo-breach-monitor.md) (lines 41-47)

- [ ] **Step 1: Replace the `safe-outputs` block**

In [.github/workflows/msdo-breach-monitor.md](../../.github/workflows/msdo-breach-monitor.md), replace this exact block:

```yaml
safe-outputs:
  noop: false
  create-issue:
    max: 1
  add-labels:
    allowed: [security-breach, supply-chain, toolchain-alert, critical, high, medium]
```

With:

```yaml
safe-outputs:
  noop:
    report-as-issue: false
  report-failure-as-issue: false
  create-issue:
    max: 1
  add-labels:
    allowed: [security-breach, supply-chain, toolchain-alert, critical, high, medium]
```

Changes: `noop: false` replaced with the `noop:\n    report-as-issue: false` object form, plus `report-failure-as-issue: false` inserted as the next key.

- [ ] **Step 2: Verify the edit**

Run:
```bash
grep -nE "noop|report-failure-as-issue|report-as-issue" .github/workflows/msdo-breach-monitor.md
```

Expected output:
```
42:  noop:
43:    report-as-issue: false
44:  report-failure-as-issue: false
```

---

## Task 4: Edit `msdo-issue-assistant.md` safe-outputs

**Files:**
- Modify: [.github/workflows/msdo-issue-assistant.md](../../.github/workflows/msdo-issue-assistant.md) (lines 32-38)

- [ ] **Step 1: Replace the `safe-outputs` block**

In [.github/workflows/msdo-issue-assistant.md](../../.github/workflows/msdo-issue-assistant.md), replace this exact block:

```yaml
safe-outputs:
  noop: false
  add-comment:
    max: 4
  add-labels:
    allowed: ["type:bug", "type:feature", "type:docs", "type:question", "type:security", "type:maintenance", "status:triage", "status:waiting-on-author", "status:repro-needed", "status:team-review", "area:action", "area:msdo-cli", "area:ci", "area:container-mapping"]
```

With:

```yaml
safe-outputs:
  noop:
    report-as-issue: false
  report-failure-as-issue: false
  add-comment:
    max: 4
  add-labels:
    allowed: ["type:bug", "type:feature", "type:docs", "type:question", "type:security", "type:maintenance", "status:triage", "status:waiting-on-author", "status:repro-needed", "status:team-review", "area:action", "area:msdo-cli", "area:ci", "area:container-mapping"]
```

Changes: `noop: false` replaced with the `noop:\n    report-as-issue: false` object form, plus `report-failure-as-issue: false` inserted as the next key.

- [ ] **Step 2: Verify the edit**

Run:
```bash
grep -nE "noop|report-failure-as-issue|report-as-issue" .github/workflows/msdo-issue-assistant.md
```

Expected output (the first three lines — additional matches will appear later in the file inside the prompt text):
```
33:  noop:
34:    report-as-issue: false
35:  report-failure-as-issue: false
```

---

## Task 5: Update `msdo-issue-assistant.md` rule 4 to call noop explicitly

**Files:**
- Modify: [.github/workflows/msdo-issue-assistant.md](../../.github/workflows/msdo-issue-assistant.md) (around lines 182-188, inside the `## Important Rules` section)

- [ ] **Step 1: Replace the rule-4 block**

Replace this exact block:

```markdown
4. **Don't respond** if:
   - The issue is not related to MSDO or security-devops-action
   - The issue is closed
   - The commenter is not the issue author (unless it's a new issue)
   - You've already responded twice and there is no new technical information in the latest user message
   - The issue has a `status:team-review` label
```

With:

```markdown
4. **Call `noop` instead of staying silent** when any of these apply. Pass a one-line reason so the decision is auditable:
   - The issue is not related to MSDO or security-devops-action
   - The issue title starts with `[aw]` or is labeled `agentic-workflows` (auto-generated failure reports, not user issues)
   - The issue is closed
   - The commenter is not the issue author (unless it's a new issue)
   - You have already responded twice and there is no new technical information in the latest user message
   - The issue has a `status:team-review` label
```

Changes: title reworded from "Don't respond" to "Call `noop` instead of staying silent"; new bullet added for `[aw]`-title / `agentic-workflows`-label issues.

- [ ] **Step 2: Verify the edit**

Run:
```bash
grep -n "Call \`noop\` instead" .github/workflows/msdo-issue-assistant.md
```

Expected: one match pointing to the rule-4 line.

---

## Task 6: Add `[aw]` example to `msdo-issue-assistant.md` "Do NOT Respond Examples"

**Files:**
- Modify: [.github/workflows/msdo-issue-assistant.md](../../.github/workflows/msdo-issue-assistant.md) (at the end of the `## Do NOT Respond Examples` section, currently ending around line 213)

- [ ] **Step 1: Append a new example at the end of the section**

Find this existing last entry in the `## Do NOT Respond Examples` section:

```markdown
**Non-author comment on existing issue:** A third party comments "I have the same problem."
→ Do not respond. The commenter is not the issue author.
```

Append **after** that block (preserve a blank line before the new entry):

```markdown

**Workflow failure issue (auto-generated):** Title starts with `[aw]` (e.g. "[aw] MSDO Issue Triage Assistant failed") or labeled `agentic-workflows`.
→ Call `noop` with reason "auto-generated failure report, not a user issue".
```

- [ ] **Step 2: Verify the edit**

Run:
```bash
grep -n "Workflow failure issue" .github/workflows/msdo-issue-assistant.md
```

Expected: one match, appearing after the `Non-author comment on existing issue` example.

Also run:
```bash
tail -5 .github/workflows/msdo-issue-assistant.md
```

Expected: the tail shows the new example as the last content in the file.

---

## Task 7: Regenerate all three lock files

**Files:**
- Modify (via compile): [.github/workflows/ci-doctor.lock.yml](../../.github/workflows/ci-doctor.lock.yml)
- Modify (via compile): [.github/workflows/msdo-breach-monitor.lock.yml](../../.github/workflows/msdo-breach-monitor.lock.yml)
- Modify (via compile): [.github/workflows/msdo-issue-assistant.lock.yml](../../.github/workflows/msdo-issue-assistant.lock.yml)

- [ ] **Step 1: Run `gh aw compile`**

Run from repo root:
```bash
gh aw compile
```

Expected: the command exits 0 and reports recompiling the three workflows. Any non-zero exit or schema error indicates the YAML edits are malformed — fix the `.md` files and retry.

- [ ] **Step 2: Inspect the lock-file diff**

Run:
```bash
git diff -- .github/workflows/*.lock.yml | head -120
```

Expected: three lock files touched. In each diff, the `frontmatter_hash` near the top of the lock file changes (because the `.md` frontmatter changed). Look for new handler wiring for the noop safe output, and the absence of a `handle_missing_safe_outputs` or similar failure-issue step (because `report-failure-as-issue: false` disables it).

If the diff shows only the `frontmatter_hash` change and no handler wiring change, the schema interpretation of `noop`/`report-failure-as-issue` may differ from expectation — pause and escalate before committing.

---

## Task 8: Commit the changes

**Files:** all six touched files in this commit.

- [ ] **Step 1: Stage all changes**

Run:
```bash
git add .github/workflows/ci-doctor.md \
        .github/workflows/ci-doctor.lock.yml \
        .github/workflows/msdo-breach-monitor.md \
        .github/workflows/msdo-breach-monitor.lock.yml \
        .github/workflows/msdo-issue-assistant.md \
        .github/workflows/msdo-issue-assistant.lock.yml
```

- [ ] **Step 2: Verify the staged diff**

Run:
```bash
git diff --cached --stat
```

Expected: six files listed, three `.md` and three `.lock.yml`.

- [ ] **Step 3: Commit with the project's oneliner style**

Run:
```bash
git commit -m "fix(ci): enable noop on agentic workflows to stop IcM page spam"
```

No Co-Authored-By line; no multi-line body.

- [ ] **Step 4: Verify the commit landed**

Run:
```bash
git log --oneline -2
```

Expected top commit: `fix(ci): enable noop on agentic workflows to stop IcM page spam`.
Second commit from top should be the earlier spec commit (`docs: add spec for agentic-workflows noop fix`).

---

## Task 9: Push the branch and open the PR

**Files:** none (GitHub operations).

- [ ] **Step 1: Push the branch**

Run:
```bash
git push -u origin fix/agentic-workflows-noop
```

Expected: branch published to `origin` with tracking configured.

- [ ] **Step 2: Create the PR**

Run (use `DimaBir` as the author account per the user's PR-account preference — if the git remote is already using that identity, a plain `gh pr create` is fine; otherwise the user handles account selection manually before this step):

```bash
gh pr create \
  --repo microsoft/security-devops-action \
  --base main \
  --head fix/agentic-workflows-noop \
  --title "fix(ci): enable noop on agentic workflows to stop IcM page spam" \
  --body "$(cat <<'EOF'
## Summary
- Enables `safe-outputs.noop` (with `report-as-issue: false`) on all three agentic workflows so the agent can explicitly signal "nothing to do" instead of exiting silent.
- Sets `safe-outputs.report-failure-as-issue: false` so edge-case silent exits no longer file `[aw] ... failed` issues that page the IcM on-call rotation.
- Updates the `msdo-issue-assistant` prompt to call `noop` in its existing "don't respond" conditions and to recognise auto-generated `[aw]` failure issues.

Fixes the false-positive failure loop documented in #247 and in [docs/superpowers/specs/2026-04-24-agentic-workflows-noop-fix-design.md](docs/superpowers/specs/2026-04-24-agentic-workflows-noop-fix-design.md).

## Test plan
- [ ] `gh aw compile` recompiles all three workflows cleanly
- [ ] `msdo-issue-assistant` negative path: post a comment on an off-topic or `[aw]`-titled issue on the PR branch — no new `[aw] ... failed` issue filed, no comment posted
- [ ] `msdo-issue-assistant` positive path: open a test issue asking a real MSDO question — bot replies normally with wiki citations and `area:msdo-cli` label
- [ ] `ci-doctor` negative path: dispatch against a successful CI run — noop, no issue filed
- [ ] `msdo-breach-monitor` negative path: `workflow_dispatch` with no new CVEs — noop, no issue filed
EOF
)"
```

No `🤖 Generated with Claude Code` footer (per user preference).

- [ ] **Step 3: Capture the PR URL**

`gh pr create` prints the PR URL on success. Record it for the validation tasks below.

---

## Task 10: Negative-path validation — `msdo-issue-assistant`

**Files:** none (exercises the PR-branch workflow).

- [ ] **Step 1: Trigger the bot against a known don't-respond case**

Option A — post a comment on issue #247 (`[aw]`-titled, will exercise the new rule):

```bash
gh issue comment 247 --repo microsoft/security-devops-action --body "test: verifying fix/agentic-workflows-noop — expect noop"
```

Option B — open a new test issue with clearly off-topic content, e.g. title "How do I deploy to AWS?" body "not MSDO-related, just testing". Close it after the run completes.

Note: the workflow runs off whatever is merged on the default branch for new issues, **unless** gh-aw activation is configured to pick up the PR head. If the run still uses the current `main` version, either (a) merge first and validate post-merge, or (b) on a fork/test repo, push the branch and re-open the same test issue. For this repo, merging first is the likely path — log this as a deliberate choice in the PR review.

- [ ] **Step 2: Observe the workflow run**

Run:
```bash
gh run list --repo microsoft/security-devops-action --workflow "MSDO Issue Triage Assistant" --limit 5
```

Expected: newest run's conclusion is `success`. Then inspect that specific run:

```bash
gh run view <RUN_ID> --repo microsoft/security-devops-action --log | grep -E "noop|safe output|agent_output|failure"
```

Expected markers:
- Evidence of the `noop` handler firing (log line referencing `noop` or `handle_noop`).
- No `"Agent succeeded but produced no safe outputs"` line.
- No step that creates or comments on a `[aw] ... failed` issue.

- [ ] **Step 3: Confirm no new `[aw]` issue was filed**

Run:
```bash
gh issue list --repo microsoft/security-devops-action --search "[aw] MSDO Issue Triage Assistant failed" --state open --limit 5
```

Expected: only the pre-existing #247 listed (or none, if it was closed). No newer `[aw]` issues.

---

## Task 11: Positive-path validation — `msdo-issue-assistant`

**Files:** none (exercises the workflow).

- [ ] **Step 1: Open a test issue with a real MSDO question**

Run:
```bash
gh issue create --repo microsoft/security-devops-action \
  --title "How do I pass --download-external-modules to checkov?" \
  --body "I want checkov (run via MSDO) to fetch external Terraform modules. How do I enable this?"
```

- [ ] **Step 2: Wait for the bot to respond (up to ~3 minutes), then inspect**

Run:
```bash
gh issue view <ISSUE_NUMBER> --repo microsoft/security-devops-action --comments
```

Expected:
- One new comment from the bot citing the wiki, mentioning `GDN_CHECKOV_DOWNLOADEXTERNALMODULES` or linking the Tool Configuration wiki page.
- The issue has the `area:msdo-cli` label applied.
- No `[aw] ... failed` issue created for this run.

- [ ] **Step 3: Close the test issue**

Run:
```bash
gh issue close <ISSUE_NUMBER> --repo microsoft/security-devops-action --comment "test issue — closing"
```

---

## Task 12: Negative-path validation — `ci-doctor`

**Files:** none.

- [ ] **Step 1: Find a successful CI run on main**

Run:
```bash
gh run list --repo microsoft/security-devops-action --workflow CI --branch main --status success --limit 3
```

Expected: at least one green CI run. Record its run ID.

- [ ] **Step 2: Manually dispatch `ci-doctor` against it (pre-merge, from the fix branch)**

Run:
```bash
gh workflow run "CI Doctor" --repo microsoft/security-devops-action --ref fix/agentic-workflows-noop
```

Using `--ref fix/agentic-workflows-noop` makes GitHub pick up the updated `.lock.yml` on the PR branch, so this exercises the fix pre-merge.

Wait ~1-2 minutes. Then:

```bash
gh run list --repo microsoft/security-devops-action --workflow "CI Doctor" --limit 3
```

Expected: newest run's conclusion is `success`.

- [ ] **Step 3: Confirm no new `[aw]` or CI Doctor diagnostic issue was filed**

Run:
```bash
gh issue list --repo microsoft/security-devops-action \
  --search "[aw] CI Doctor failed OR [CI Doctor]" \
  --state open --limit 5
```

Expected: no newer entries than the pre-existing baseline. If CI Doctor found nothing new to diagnose (green run), it must have noop'd cleanly.

---

## Task 13: Negative-path validation — `msdo-breach-monitor`

**Files:** none.

- [ ] **Step 1: Dispatch the monitor (pre-merge, from the fix branch)**

Run:
```bash
gh workflow run "MSDO Toolchain Breach Monitor" --repo microsoft/security-devops-action --ref fix/agentic-workflows-noop
```

Using `--ref fix/agentic-workflows-noop` makes GitHub pick up the updated `.lock.yml` on the PR branch.

- [ ] **Step 2: Observe the workflow run**

Run:
```bash
gh run list --repo microsoft/security-devops-action --workflow "MSDO Toolchain Breach Monitor" --limit 3
```

Expected: newest run's conclusion is `success`.

Inspect the log for the noop call:

```bash
gh run view <RUN_ID> --repo microsoft/security-devops-action --log | grep -E "noop|no new incidents|toolchain-alert"
```

Expected: evidence of a noop call (unless a genuine CVE in the window would produce a `toolchain-alert` issue — which is a positive-path outcome, not a failure).

- [ ] **Step 3: Confirm no new `[aw] MSDO Toolchain Breach Monitor failed` issue was filed**

Run:
```bash
gh issue list --repo microsoft/security-devops-action \
  --search "[aw] MSDO Toolchain Breach Monitor failed" \
  --state open --limit 5
```

Expected: no newer entries.

---

## Task 14: Complete the PR

**Files:** none (GitHub).

- [ ] **Step 1: Tick the PR's Test plan checkboxes**

In the PR description, tick each checkbox that the validation tasks confirmed.

Run:
```bash
gh pr view <PR_NUMBER> --repo microsoft/security-devops-action
```

Edit description via:
```bash
gh pr edit <PR_NUMBER> --repo microsoft/security-devops-action --body "<updated body with checkboxes ticked>"
```

- [ ] **Step 2: Hand off for human review and merge**

The PR is now complete. Post a short summary comment and leave the merge to the repository maintainer per normal review process. The user will close #247 manually after merge.
