# ai-dev-loop-engine

Reusable GitHub Actions workflow and prompt set for a shared AI development loop.

The intended architecture is:

1. This engine repository contains only reusable workflow logic and AI prompts.
2. Each business repository, such as a plugin, website, or mini-program, contains only a very small trigger workflow.
3. A user comments `/aidev` on an issue in the business repository.
4. GPT reads the issue plus trigger comment and generates a structured plan.
5. Codex implements only that plan and must not freely expand scope.
6. Codex commits to one fixed branch and one PR for the issue.
7. GPT reviews `plan.md`, changed files, diff, and test result.
8. If review does not pass, the workflow generates a revised plan and lets Codex fix only the reviewer findings.
9. Every single workflow run is bounded by `max_iterations`.
10. The workflow never auto-merges. A human must review and merge the PR.

## What It Does

- Reads an issue and the `/aidev` trigger comment from the caller repository.
- Checks out this engine repository explicitly for reusable prompts.
- Generates a structured plan at `.ai-dev/issue-<issue_number>/plan.md`.
- Creates or updates one fixed branch per issue: `ai/issue-<issue_number>` by default.
- Creates a PR only when no open PR exists for that issue branch.
- Reuses the same open PR for reruns, revised plans, and `/aidev continue` comments.
- Runs Codex against the generated plan only.
- Runs `git status` after Codex changes.
- Runs the configured `test_command`, when provided.
- Sends GPT review input that includes `plan.md`, changed files, diff, and test result.
- Stops after `max_iterations`, defaulting to `3`.

This is intentionally not an infinite loop. If a run reaches `max_iterations` before review passes, comment `/aidev continue` on the same issue after reviewing the PR state. The next run continues on the same branch and same PR.

## Required Secret

Add this secret in each repository that calls the engine:

- `OPENAI_API_KEY`

In GitHub, go to `Settings` -> `Secrets and variables` -> `Actions` -> `New repository secret`.

Do not put API keys in issues, PRs, comments, workflow inputs, source code, generated files, or logs.

## Engine Workflow

This repository provides the reusable workflow:

```yaml
.github/workflows/ai-dev-loop.yml
```

It is called with `workflow_call` and accepts:

| Input | Default | Description |
| --- | --- | --- |
| `issue_number` | required | Issue number to implement. |
| `trigger_comment` | `""` | Comment text that triggered the loop. |
| `base_branch` | `main` | Branch used when creating a new issue branch. |
| `branch_prefix` | `ai/issue-` | Prefix for the fixed issue branch. |
| `engine_repository` | `yh3970/ai-dev-loop-engine` | Repository to checkout for prompts. |
| `engine_ref` | `main` | Ref to checkout from the engine repository. |
| `model` | `gpt-4.1` | OpenAI model used by planner and reviewer calls. |
| `install_command` | `""` | Optional install command run before implementation. |
| `test_command` | `""` | Optional test command run after Codex implementation. |
| `working_directory` | `.` | Directory where install and test commands run. |
| `max_iterations` | `3` | Maximum review/fix iterations for one workflow run. |

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

## Recommended Trigger Workflow

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
      startsWith(github.event.comment.body, '/aidev') &&
      contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association)
    uses: yh3970/ai-dev-loop-engine/.github/workflows/ai-dev-loop.yml@main
    with:
      issue_number: ${{ github.event.issue.number }}
      trigger_comment: ${{ github.event.comment.body }}
      base_branch: main
      branch_prefix: ai/issue-
      engine_repository: yh3970/ai-dev-loop-engine
      engine_ref: main
      model: gpt-4.1
      install_command: npm ci
      test_command: npm test
      working_directory: .
      max_iterations: 3
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

The authorization guard is important. Do not allow arbitrary internet users to trigger AI code-writing workflows. The recommended trigger allows `/aidev` only for issue comments by users whose `author_association` is `OWNER`, `MEMBER`, or `COLLABORATOR`, and it ignores comments on pull requests.

If a project has no install or test command, omit `install_command` and `test_command` rather than inventing unsafe commands.

## Using `/aidev`

Open an issue in the project repository and describe the requested change clearly. Then add a comment:

```text
/aidev
```

You can include extra scoped instructions after the command:

```text
/aidev please keep the change limited to the settings page and add tests
```

If review does not pass before `max_iterations`, inspect the PR and then continue on the same fixed branch and PR with:

```text
/aidev continue
```

The workflow will create or update:

- branch `ai/issue-<issue_number>`
- the same open PR for that branch
- `.ai-dev/issue-<issue_number>/plan.md` on the issue branch

The same issue always maps to the same branch and PR, so reruns and revised plans do not create duplicate PRs.

## PR and Branch Stability

The reusable workflow is designed so that each issue maps to one fixed branch:

```text
${branch_prefix}${issue_number}
```

PR behavior is stable:

- If the branch already exists, the workflow checks out that branch.
- If an open PR already exists for that branch, the workflow updates that PR.
- If no open PR exists for that branch, the workflow creates a PR.
- Reruns, `/aidev continue`, and revised plans keep using the same branch and PR.
- The workflow does not merge or auto-merge the PR.

## Safety Notes

- The workflow never auto-merges; final merge requires human review.
- Trigger workflows should restrict `/aidev` to `OWNER`, `MEMBER`, or `COLLABORATOR` comments.
- The reusable workflow uses concurrency per repository and issue so duplicate `/aidev` comments do not run against the same branch and PR at the same time.
- Keep `max_iterations` finite; use `3` unless there is a strong reason to change it.
- Use `/aidev continue` for another bounded run instead of configuring an unbounded loop.
- Keep `OPENAI_API_KEY` only in GitHub Actions secrets.
- Do not paste secrets into issues, PRs, comments, code, plans, or logs.
- Review the generated PR and tests before merging.
