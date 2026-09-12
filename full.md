---
name: Repo Assist
description: Daily repository maintenance with Copilot, reviewed pull requests, and GitHub issue memory.
on:
  schedule:
    - cron: "30 7 * * *"
      timezone: Australia/Melbourne
  workflow_dispatch:
  permissions:
    contents: read
    pull-requests: read
    actions: read
  steps:
    - id: check
      name: Check authentication, backlog, and duplicate installations
      env:
        GH_TOKEN: ${{ github.token }}
      run: |
        if ! gh auth status; then
          echo "Repo Assist: gh CLI is not authenticated in this environment. No action taken."
          exit 1
        fi
        gh repo view "$GITHUB_REPOSITORY" --json nameWithOwner,defaultBranchRef,visibility
        count=$(gh pr list --repo "$GITHUB_REPOSITORY" --state open --limit 8 \
          --label repo-assist --search 'in:title "[repo-assist]"' --json number --jq 'length')
        if [[ "$count" -ge 8 ]]; then
          echo "Repo Assist: 8 or more open [repo-assist] PRs. Skipping this run to avoid backlog."
          exit 1
        fi
        duplicates=$(gh workflow list --repo "$GITHUB_REPOSITORY" --all --limit 1000 \
          --json name,path,state --jq '[.[] | select(.state == "active" and
          ((.path | startswith(".github/workflows/repo-assist")) or
          (.name | test("repo[ _-]?assist"; "i"))) and
          .path != ".github/workflows/repo-assist.lock.yml")] | length')
        if [[ "$duplicates" -gt 0 ]]; then
          echo "Repo Assist: another Repo Assist workflow is enabled. No action taken."
          exit 1
        fi
if: needs.pre_activation.outputs.check_result == 'success'
concurrency:
  group: repo-assist
  cancel-in-progress: false
features:
  group-concurrency-queue: false
engine: copilot
runs-on: ubuntu-24.04
runs-on-slim: ubuntu-24.04
timeout-minutes: 60
permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read
  checks: read
  statuses: read
network:
  allowed: [defaults, github, node, python, go]
checkout:
  fetch: ["*"]
  fetch-depth: 0
runtimes:
  node:
    version: "24"
  python:
    version: "3.14.7"
  go:
    version: "1.26"
tools:
  github:
    mode: gh-proxy
    github-token: ${{ secrets.GITHUB_TOKEN }}
    min-integrity: none
  bash: true
