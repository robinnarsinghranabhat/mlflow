# Code Review Request

Review the changes for RHOAIENG-56609: settling the `RequiredResource` API shape for scorer-declared permissions.

## Setup

Review against a clean diff — not the working directory (which has unrelated scratch files). Use the commit diff:

```bash
git diff HEAD~1..HEAD
```

Or if using Claude Code's Agent tool, use `isolation: "worktree"` to get a clean copy of the repo with only committed files.

The branch is `jobs-execution-rfc2`. The commit to review is the latest: `feat: Add RequiredResource API shape for scorer-declared permissions`.

## Before reviewing

1. **Read the PR write-up first:** `RESOURCE_CLASS_PR.md`
2. **Read the RFC discussion thread** — this is the debate this PR resolves. Fetch and read:
   - https://github.com/mlflow/rfcs/pull/2#discussion_r3012870624
3. **Read the relevant RFC sections** from `huey_research/0002-job-executor-plugins.md`:
   - Lines 779-841: Permission extraction and the original `RequiredResource` proposal
   - Lines 756-777: Permission model and privilege escalation checks
   - Lines 843-866: API server enforcement table
4. **If you need Jira ticket context**, use the `/acli-jira` skill:
   - `RHOAIENG-56609` — this ticket
   - `RHOAIENG-56613` — downstream: permission extraction (primary consumer of this shape)
   - `RHOAIENG-56598` — downstream: per-job executor routing

## Changed/new source files

- `mlflow/entities/_required_resource.py` (new)
- `mlflow/genai/scorers/resources.py` (new, re-export)
- `mlflow/genai/scorers/base.py` (modified)
- `mlflow/genai/scorers/__init__.py` (modified)
- `tests/genai/scorers/test_resources.py` (new)

## Existing code for comparison

- `mlflow/models/resources.py` — the existing `Resource` ABC (Databricks model serving)
- `mlflow/server/auth/permissions.py` — RBAC resource type constants our `type` values must match

## What to look for

1. Does the `RequiredResource` shape work for downstream permission extraction (RHOAIENG-56613)?
2. Is the decision not to reuse `mlflow.models.resources.Resource` well-justified?
3. Are the serialization round-trips correct (scorer → JSON → DB → JSON → scorer)?
4. Are there edge cases the tests miss?
5. Is the placement in `mlflow/entities/` appropriate, or should it live elsewhere?
