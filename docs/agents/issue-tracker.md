# Issue tracker: GitHub + Jira (hybrid)

This repo uses two issue trackers:

- **GitHub Issues** on `ovn-kubernetes/ovn-kubernetes` for upstream bugs and community contributions
- **Jira** for OpenShift-scoped work:
  - **OCPBUGS** — bug tracking
  - **CORENET** — feature work

## GitHub conventions

Use the `gh` CLI for all GitHub operations.

- **Create an issue**: `gh issue create --title "..." --body "..."`. Use a heredoc for multi-line bodies.
- **Read an issue**: `gh issue view <number> --comments`, filtering comments by `jq` and also fetching labels.
- **List issues**: `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] | {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'` with appropriate `--label` and `--state` filters.
- **Comment on an issue**: `gh issue comment <number> --body "..."`
- **Apply / remove labels**: `gh issue edit <number> --add-label "..."` / `--remove-label "..."`
- **Close**: `gh issue close <number> --comment "..."`

Infer the repo from `git remote -v` — `gh` does this automatically when run inside a clone.

## Jira conventions

Use the Atlassian MCP tools for Jira operations. The cloud ID can be resolved via `getAccessibleAtlassianResources`.

- **OCPBUGS** — use for bugs filed against OpenShift OVN-Kubernetes
- **CORENET** — use for feature work, enhancements, and planning

When Jira MCP commands fail due to permissions, immediately report the failure and provide the content in a copyable format so the user can post manually. Do not retry repeatedly.

## Pull requests as a triage surface

**PRs as a request surface: no.** External PRs are reviewed through the normal code review process, not triaged alongside issues.

## When a skill says "publish to the issue tracker"

Default to creating a GitHub issue unless the context is clearly OpenShift-scoped (bug → OCPBUGS, feature → CORENET), in which case create a Jira issue.

## When a skill says "fetch the relevant ticket"

- If the reference looks like `#123` or a plain number, run `gh issue view <number> --comments`.
- If it looks like `OCPBUGS-12345` or `CORENET-123`, use the Jira MCP `getJiraIssue` tool.
