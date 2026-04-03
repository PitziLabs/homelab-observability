---
name: Implementation task
about: Dispatch an implementation task to Claude Code
title: ""
labels: ["automation"]
assignees: ""
---

## What to implement

<!-- Describe the change in plain English. Be specific about which files,
     which behavior, and what the end state looks like. -->


## Acceptance criteria

<!-- Checklist of what "done" looks like. Claude Code uses this to verify. -->

- [ ]

## Constraints

<!-- Anything Claude Code should NOT do, or boundaries to stay within. -->

- Do not add Docker services without discussion
- Do not hardcode secrets — use environment variables via .env
- Dashboard JSON must use provisioning format (no __inputs/__requires, schema v39, UID pattern firewalla-<n>)
- Keep Loki label cardinality low
- Do not modify unrelated dashboards or configs

## Auto-merge

When implementation is complete:
1. Validate Docker Compose config: `docker compose config --quiet`
2. If shell scripts were modified: `shellcheck --severity=warning --shell=bash <script>`
3. Create a PR with a clear title and description referencing this issue
4. Enable auto-merge: `gh pr merge --auto --squash`

@claude implement this
