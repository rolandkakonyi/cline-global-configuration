---
description: Guidelines for creating pull requests including template usage, issue linking, and draft requirements.
author: https://github.com/rolandkakonyi
version: 1.0
globs: ["**/*"]
tags: ["pull-request", "github", "workflow", "templates", "best-practices"]
---

1. If a pull request template exists in .github/PULL_REQUEST_TEMPLATE.md file, always follow it! Check first for file existence.
2. If the template has a line like "Issue: https://bitmovin.atlassian.net/browse/<PROJECT_ID>-<ISSUE_NUMBER>", replace "<PROJECT_ID>-<ISSUE_NUMBER>" (e.g. PI-4155) from the current git branch name by fining the pattern of "PROJECT_ID-ISSUE_NUMBER" after the prefix "feature/" in the branch name. If unsure about the project prefix, ask for user input. If there is no atlassian reference, ignore this istruction regarding "Issue: "
3. Pull requests should ALWAYS BE CREATED AS A DRAFT unless explicitly asked otherwise!!!
