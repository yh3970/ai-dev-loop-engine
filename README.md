# ai-dev-loop-engine

Reusable GitHub Actions workflow for an AI development loop:

Issue -> GPT plan -> Codex implementation -> fixed PR branch -> GPT review -> revised plan and fix loop when needed.

The engine is designed to be called from small trigger workflows in other repositories. It does not auto-merge PRs.

## What It Does

- Reads an issue and the `/aidev` trigger comment.
- Generates a structured `plan.md`.
- Creates or updates one fixed branch per issue: `ai/issue-<issue_number>`.
- Creates a PR if none exists for that branch.
- Reuses the same open PR for later runs on the same issue.
- Runs Codex against `plan.md` only.
- Commits and pushes Codex changes to the issue branch.
- Reviews the diff against `plan.md`.
- If review returns `needs_changes`, generates a revised plan and asks Codex to fix it.
- Stops after `max_iterations`, defaulting to `3`.

Infinite loops are not recommended. Keep `max_iterations` small; the default is `3`.

## Required Secret

Add this secret in each repository that calls the engine:

- `OPENAI_API_KEY`

In GitHub, go to `Settings` -> `Secrets and variables` -> `Actions` -> `New repository secret`.

Do not put API keys in issues, PRs, comments, workflow inputs, source code, or generated files.

## Engine Workflow

This repository provides the reusable workflow:

```yaml
.github/workflows/ai-dev-loop.yml
```

It is called with `workflow_call` and accepts:

- `issue_number`
- `trigger_comment`
- `base_branch`
- `branch_prefix`
- `max_iterations`

It requires:

- `OPENAI_API_KEY`

The default branch rule is:

```text
branch = ${branch_prefix}${issue_number}
```

With the recommended defaults, issue `123` uses:

```text
ai/issue-123
```

If that branch already exists, the workflow checks it out and continues. If not, it creates the branch from `base_branch`.

## Trigger Workflow Example

Add a small workflow to each project repository that should use this engine:

```yaml
name: Trigger AI Dev Loop

on:
  issue_comment:
    types: [created]

permissions:
  contents: write
  pull-requests: write
  issues: write

jobs:
  trigger:
    if: >
      github.event.issue.pull_request == null &&
      startsWith(github.event.comment.body, '/aidev')
    uses: YOUR_ORG_OR_USER/ai-dev-loop-engine/.github/workflows/ai-dev-loop.yml@main
    with:
      issue_number: ${{ github.event.issue.number }}
      trigger_comment: ${{ github.event.comment.body }}
      base_branch: main
      branch_prefix: ai/issue-
      max_iterations: 3
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

Replace `YOUR_ORG_OR_USER` with the owner of this engine repository.

## Using `/aidev`

Open an issue in the project repository and describe the requested change clearly. Then add a comment:

```text
/aidev
```

You can include extra scoped instructions after the command:

```text
/aidev please keep the change limited to the settings page and add tests
```

The workflow will create or update:

- branch `ai/issue-<issue_number>`
- the same open PR for that branch
- `plan.md` on the issue branch

The same issue always maps to the same branch and PR, so rerunning `/aidev` continues the existing work instead of creating duplicate PRs.

## Safety Notes

- The workflow never auto-merges.
- Keep `max_iterations` finite; use `3` unless there is a strong reason to change it.
- Review the generated PR before merging.
- Keep `OPENAI_API_KEY` only in GitHub Actions secrets.
- Do not paste secrets into issues, PRs, comments, code, or plans.
