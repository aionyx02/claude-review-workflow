---
name: setup-review-pipeline
description: Scaffold a layered code review pipeline (local verification, Claude PR review, security review, merge ruleset) into the current GitHub repo, adapted for a personal account.
disable-model-invocation: true
---

# Setup Review Pipeline

Based on "How the Claude Code team does code review" (claude.dev, 2026-10-15): review is layers, not one gate. Machines do most of the reading; human attention goes where the risk is; fast rollback catches the rest.

## Decisions already made (don't re-litigate)

The user is a solo developer on a **personal** GitHub account. The article assumes a Team/Enterprise org, so:

| Article | Here | Why |
|---|---|---|
| Code Review for Claude Code (managed) | `anthropics/claude-code-action` with our own prompt | Managed product is Team/Enterprise only |
| `claude-code-security-review` action | Second job on `claude-code-action` with a security prompt | That action only accepts an API key (checked 2026-10-26); we use OAuth |
| 1-2 human approvals by risk tier | **No required approvals** | GitHub won't let you approve your own PR; it would deadlock |
| Open findings block merge | Ruleset: required review-thread resolution | Every finding is fixed or resolved with a reason |
| CODEOWNERS for sensitive paths | "Always review deeply" list in `REVIEW.md` | One person; CODEOWNERS adds nothing |
| Merge queue | `strict_required_status_checks_policy` (branch must be up to date) | Merge queue isn't available on personal repos |

- **Auth**: `CLAUDE_CODE_OAUTH_TOKEN` (subscription). Switching to `ANTHROPIC_API_KEY` later is a one-line change in the workflow.
- **Never** use `pull_request_target`. Fork PRs get no secrets, so the review jobs fail on them on purpose: a skipped job would satisfy the required check with no review.

## Steps

### 1. Preflight

- `gh auth status`; `gh repo view --json nameWithOwner,owner,visibility,defaultBranchRef,isInOrganization`
- `isInOrganization` true: stop and tell the user that CODEOWNERS, org secrets and merge queue become available, so the table above should be revisited.
- Private repo: Actions minutes are limited, so mention that security review can be restricted to `opened`/`ready_for_review`. Rulesets on a private personal repo are **only enforced with GitHub Pro**; on Free, step 9 does nothing and the gate is advisory. Say so before going on.

### 2. Pick the tier

Ask the user, with a recommendation based on what you see:

- **Full**: maintained, has users, releases, or CI/tests already. Gets everything below.
- **Light**: coursework, practice, notebooks, throwaway. Gets step 3 only; nothing on GitHub changes.

### 3. Local layer (both tiers)

Add to the project's `CLAUDE.md` (create it if missing, keep it short), filling in the real commands you detect:

```markdown
## Verification
- Test: `<cmd>`
- Lint / format: `<cmd>`
- Build: `<cmd>`

Show the command output before saying a change is done.

## Before opening a PR
Run the checks above, then `/code-review` against the base branch and `/security-review`.
```

Full tier also appends: "Fill the PR template; whoever asked for the change signs off on Testing and How to revert."

Done when `CLAUDE.md` names commands that actually run: execute each one once and show the output.

Optionally offer a formatter hook in the project's `.claude/settings.json` (PostToolUse on `Edit|Write`) matching the project's formatter. Python projects use uv + ruff. Light tier stops here.

### 4. CI (full tier)

Find existing workflows in `.github/workflows/` and record the **job names** (they become required status checks). If there is no CI, propose a minimal one for the stack (Python: `uv sync`, `uv run ruff check`, `uv run pytest`) and confirm before adding it.

### 5. Refresh action versions

Templates go stale. Before copying, check the current major tags:

```bash
gh api repos/anthropics/claude-code-action/releases/latest -q .tag_name
gh api repos/actions/checkout/releases/latest -q .tag_name
```

Use the major tag (`v1`, `v7`, ...), and skim `action.yml` of `claude-code-action` if its major changed, because input names may have moved.

### 6. Write the files

From `templates/` in this skill's directory:

| Template | Destination | Notes |
|---|---|---|
| `claude-review.yml` | `.github/workflows/claude-review.yml` | Update action versions from step 5 |
| `REVIEW.md` | `REVIEW.md` | **Fill it in** from the codebase: what failure looks like, deep paths, skip paths, review language |
| `pull_request_template.md` | `.github/pull_request_template.md` | If one exists, merge in the missing sections instead of overwriting |
| `ruleset.json` | not committed | Used in step 9 |

Show the user every file before moving on.

### 7. Secret (the user does this)

Never ask for the token in chat. Tell the user to run, **in their own terminal**:

```bash
claude setup-token
gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo <owner>/<repo>
```

`gh secret set` prompts for the value, so it never lands in a transcript. One token works for every repo; it lasts about a year, so suggest a calendar reminder.

### 8. Smoke test

Ask before pushing. Then create a branch, commit, push, and open a PR. That PR is the test: confirm with `gh pr checks` that `claude-review` and `security-review` both ran and each posted a summary comment. If a job fails, read the log with `gh run view --log-failed` before changing anything.

### 9. Ruleset (after the checks have reported once)

Required status checks only match contexts GitHub has seen, which is why this runs after step 8.

1. Add the CI job names from step 4 to `required_status_checks` in `templates/ruleset.json` and write it to the scratchpad. Show it to the user and ask before applying, because it changes how the repo accepts pushes.
2. `gh api repos/<owner>/<repo>/rulesets` to look for an existing `review-gate`:
   - none: `gh api repos/<owner>/<repo>/rulesets --method POST --input <file>`
   - exists: `gh api repos/<owner>/<repo>/rulesets/<id> --method PUT --input <file>`
3. Verify that the smoke-test PR shows the rules as merge requirements.

The repo-admin bypass is `pull_request` mode: the admin can merge a PR past the gate (an escape hatch for an expired token or a broken reviewer) but can't push straight to the default branch.

### 10. Hand off

Tell the user:
- When the reviewer keeps flagging the same thing, fix it at the source: `REVIEW.md`, a lint rule, or `CLAUDE.md`.
- Every inline finding blocks the merge until it is fixed or its thread is resolved. Resolving a Nit with a one-line reason is the expected path.
- Rounds 1-2 are full reviews; from round 3 the reviewer reads only new changes; after three rounds it posts only Important findings and asks the author to rethink the approach.
- Start with this one repo and tune `REVIEW.md` for a few weeks before rolling out to others.
- Escape hatch: the admin bypass above (also the way to merge a fork PR). Token expiry looks like both jobs failing at auth.
