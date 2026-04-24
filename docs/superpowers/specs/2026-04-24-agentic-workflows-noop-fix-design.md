# Agentic Workflows — `noop` Fix Design

**Date:** 2026-04-24
**Tracking issue:** [#247 — [aw] MSDO Issue Triage Assistant failed](https://github.com/microsoft/security-devops-action/issues/247)

## Problem

Three agentic workflows in `.github/workflows/` each set `safe-outputs.noop: false`
while their prompts instruct the agent to call `noop` or stay silent under
various conditions:

| Workflow | File | Silent-path trigger |
|---|---|---|
| MSDO Issue Triage Assistant | `msdo-issue-assistant.md` | "Don't respond if" rules (off-topic issue, closed, non-author, `status:team-review`, already-responded, etc.) |
| CI Doctor | `ci-doctor.md` | "If the workflow succeeded, do nothing (noop)"; duplicate-issue check |
| MSDO Toolchain Breach Monitor | `msdo-breach-monitor.md` | "Call `noop` with a one-line summary" when no new CVEs |

With `noop` disabled, the agent has no way to signal "intentional no-op." gh-aw
reads no `agent_output.json`, treats the run as failure, and files an issue
titled `[aw] <workflow> failed`.

In this repository, **every new GitHub issue opens a CRI IcM ticket that pages
on-call**. Each false-positive failure issue is therefore a human page. For
`msdo-issue-assistant`, the failure issue itself (#247) carries the
`agentic-workflows` label and re-triggers the bot on every comment, producing
a self-sustaining spam loop.

Evidence from run `24783399971`:

```
Agent conclusion: success
Error reading agent output file: ENOENT: no such file or directory, open '/tmp/gh-aw/agent_output.json'
Agent succeeded but produced no safe outputs
Found existing issue #247: https://github.com/microsoft/security-devops-action/issues/247
Added comment to existing issue #247
```

## Goals

1. Stop paging IcM on-call for false-positive agent failures.
2. Let each bot explicitly signal "nothing to do" when its prompt says to.
3. Preserve normal behaviour on genuine user questions and genuine incidents.

## Non-goals

- Closing issue #247 (user closes manually after merge).
- Adding non-paging alerting for real agent failures (future work).
- Fixing the unrelated broken `ContainerMapping` tests on main.
- Changes to non-agentic workflows.

## Approach

Hybrid fix — enable `noop` as the root-cause fix, add `report-failure-as-issue: false`
as a safety net so any edge case that still produces no output never pages IcM.

### Changes to `safe-outputs` (all three `.md` files)

```yaml
safe-outputs:
  noop: true                      # was: false
  report-failure-as-issue: false  # new
  # ... existing keys (add-comment, add-labels, create-issue, etc.) unchanged
```

Semantics:
- `noop: true` registers the `noop` safe-output handler so the agent can call it
  with a reason. gh-aw records a successful no-op run and does not treat it as
  failure.
- `report-failure-as-issue: false` prevents gh-aw from filing an issue when the
  run ends in failure or with no outputs. Genuine failures remain visible in the
  Actions tab.

### Additional edits in `msdo-issue-assistant.md` only

The prompt currently uses `## Important Rules → Don't respond if` and
`## Do NOT Respond Examples`. Update both to direct the agent to call `noop`
explicitly.

**Rule replacement** (replace the existing rule 4 "Don't respond if" block):

```markdown
4. **Call `noop` instead of staying silent** when any of these apply. Pass a
   one-line reason so the decision is auditable:
   - The issue is not related to MSDO or security-devops-action
   - The issue title starts with `[aw]` or is labeled `agentic-workflows`
     (auto-generated failure reports, not user issues)
   - The issue is closed
   - The commenter is not the issue author (unless it's a new issue)
   - You have already responded twice and there is no new technical
     information in the latest user message
   - The issue has a `status:team-review` label
```

**New entry in "Do NOT Respond Examples"** (append):

```markdown
**Workflow failure issue (auto-generated):** Title starts with `[aw]`
(e.g. "[aw] MSDO Issue Triage Assistant failed") or labeled
`agentic-workflows`.
→ Call `noop` with reason "auto-generated failure report, not a user issue".
```

No prompt edits in `ci-doctor.md` or `msdo-breach-monitor.md` — their prompts
already say "call noop" / "do nothing (noop)" and will work correctly once
`noop: true` is set.

### Lock-file regeneration

After `.md` edits, run `gh aw compile` locally (gh-aw CLI v0.61.0,
matching the version recorded in the existing lock-file header) to
regenerate the three `.lock.yml` files. Both `.md` and `.lock.yml` go in
the same PR so reviewers can diff intent against generated output.

## Validation

Existing unit tests on main are broken (ContainerMapping) and do not cover
agentic-workflow behaviour. Validation is behavioural, via `workflow_dispatch`
runs on the PR branch:

1. **Compile check:** `gh aw compile` succeeds without error; lock-file diff
   contains the expected `noop` handler wiring and no other unintended changes.
2. **`msdo-issue-assistant` negative path:** on the PR branch, post a comment
   on an existing off-topic issue or on issue #247 itself (this fires the
   `issue_comment: created` trigger against the PR-branch workflow via the
   normal gh-aw activation flow). Expect the run to succeed, no new comment
   posted, no new `[aw] ... failed` issue filed.
3. **`msdo-issue-assistant` positive path:** open a test issue with a real MSDO
   question (for example "how do I pass `--download-external-modules` to
   checkov?"). Expect the bot to reply normally, citing the wiki, applying the
   `area:msdo-cli` label.
4. **`ci-doctor` negative path:** trigger a CI run that succeeds on `main` or a
   `release/**` branch (the workflow auto-fires on `workflow_run: CI completed`),
   or dispatch manually and point it at a successful run. Expect noop, no
   issue filed.
5. **`msdo-breach-monitor` negative path:** `workflow_dispatch` manually when
   no new CVEs are in the advisory window. Expect noop, no issue filed.

If any dry run still files a `[aw] ... failed` issue, the safety net
(`report-failure-as-issue: false`) has not taken effect — investigate before
merging.

## Rollout

- One PR on branch `fix/agentic-workflows-noop`, base `main`.
- PR title: `fix(ci): enable noop on agentic workflows to stop IcM page spam`.
- Merge once dry-run validation passes.
- User closes #247 manually after merge.

## Risks

| Risk | Mitigation |
|---|---|
| `report-failure-as-issue: false` hides a real agent failure | Accepted trade-off — false positives page IcM; real failures remain in Actions tab and can be wired to non-paging alerts later |
| gh-aw v0.61.0 interprets `noop: true` differently than expected | Lock-file diff is reviewed before merge; fall back to `report-failure-as-issue: false` only (Approach 2) if the generated handler looks wrong |
| Prompt edits cause `msdo-issue-assistant` to noop on cases users want a reply on | Conditions are identical to existing "Don't respond" rules — behaviour unchanged, only the exit mechanism becomes explicit. Positive-path dry run catches regressions |
