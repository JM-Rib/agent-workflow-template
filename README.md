# agent-workflow-template

GitHub template with integrated workflows for implementing agent-ready issues and
processing pull request review feedback with GitHub Copilot CLI.

## Requirements

- Add a repository secret named `COPILOT_TOKEN`. It is used to authenticate the
  Copilot CLI and must have the `Copilot Requests` permission.
- Refine an issue or pull request before applying the `agent-ready` label.
- Keep implementation pull requests in the same repository; the PR follow-up
  workflow does not process fork branches.

Both workflows run on `ubuntu-latest`, use the repository `GITHUB_TOKEN` for
GitHub API operations, and serialize runs with concurrency groups. Copilot is
not given a write-capable GitHub credential; the workflow validates and publishes
its output afterward.

## Issue implementation

**Workflow:** `.github/workflows/copilot-issues.yml`
**Name:** `Implement agent-ready issue`

### How it starts

- Automatically when an issue receives the `agent-ready` label.
- Manually through **Actions → Implement agent-ready issue → Run workflow**.
  An optional issue number can target one specific open issue.

The selected issue must be open, must not be a pull request, must have the
`agent-ready` label, and must be authored by a repository owner, member,
collaborator, or `github-actions[bot]`. Without a manual issue number, all
eligible open issues are selected in creation order and processed as a matrix.

### What it does

1. Verifies `COPILOT_TOKEN` and changes `agent-ready` to `agent-working`.
2. Snapshots the issue body and trusted comments into a temporary `.copilot/`
   directory.
3. Runs Copilot CLI to implement the issue, following `AGENT.md` and the
   repository instructions.
4. If clarification is required, posts the generated question, changes the
   issue to `needs-refinement`, and makes no tracked-file changes.
5. Otherwise, creates `copilot/issue-<number>`, commits and pushes the changes,
   and opens a pull request using the generated title and body.
6. On failure, returns the issue to `agent-ready` unless a pull request was
   already created.

Copilot is instructed not to modify `.github/`, add dependencies, use GitHub
credentials, or commit changes itself. The publish step also rejects changes
under `.github/`.

## Pull request review follow-ups

**Workflow:** `.github/workflows/copilot-pr-followups.yml`
**Name:** `Address PR review feedback`

### How it starts

- Automatically when a pull request receives the `agent-ready` label.
- Manually through **Actions → Address PR review feedback → Run workflow**.
  An optional pull request number can target one specific PR.

The PR must be open, labeled `agent-ready`, and have a branch in the same
repository. It is processed only when the repository owner has submitted new
review feedback since the workflow's last marker. PRs already labeled
`agent-working` are skipped.

### What it does

1. Changes `agent-ready` to `agent-working` and snapshots the PR, diff, and
   owner reviews/comments.
2. Uses the latest owner review state to choose a mode:
   - `CHANGES_REQUESTED`: Copilot may update the PR branch.
   - `APPROVED` or `COMMENTED`: read-only; Copilot may only prepare explanations.
3. Validates the generated summary, threaded review replies, and any explicitly
   requested follow-up issues.
4. Pushes code changes to the existing PR branch when allowed, replies to
   inline comments, creates requested follow-up issues, and posts a summary.
5. Marks the PR `agent-done` after success; on failure it removes
   `agent-working` so it can be retried.

Follow-up issues receive the `follow-up` label and must be refined before adding
`agent-ready`. Copilot may not modify `.github/`, create issues on its own
initiative, or change code when the owner has not requested changes.

## Labels used

| Label | Meaning |
| --- | --- |
| `agent-ready` | Queued for the corresponding workflow |
| `agent-working` | Currently being processed |
| `needs-refinement` | Issue needs clarification before implementation |
| `agent-done` | Latest PR owner feedback was processed |
| `follow-up` | Issue created explicitly from a PR review |
