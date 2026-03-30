---
name: list-core-prs
description: >
  List open pull requests from ROCm/rocm-libraries that are labeled "project: composablekernel"
  and focused on ck_tile core - a folder contains every basic functions and structures to create a GPU kernel using ck_tile.
  Use this skill whenever the user create pull ck_tile core requests in progress on the rocm-libraries repo. Also trigger when the user asks things like
  "what PRs touch the core", "show me open core changes", "list core pull requests",
  "what's being worked on for core in CK_TILE", "core unit test PRs", or "core CI changes".
---

# List Core-Related PRs in ROCm/rocm-libraries

Find and present all open pull requests in `ROCm/rocm-libraries` labeled `"project: composablekernel"` that primarily modify files under `ck_tile/core/`.

## Prerequisites

- **`gh` CLI**: This skill requires the [GitHub CLI](https://cli.github.com/) to be installed and authenticated. All PR fetching, file listing, and diff inspection is done via `gh api`, `gh pr view`, and `gh pr diff` bash commands.


## Why this skill exists

PR titles in this repo are often vague or misleading — a PR titled "Enable MXFP6 for MX GEMM op" might change ck tile core to add new data format. Titles alone are not reliable for filtering. This skill reads diffs and changed file paths to determine the real focus of each PR.


## Step 1: Fetch all labeled PRs

Use `gh api` to fetch all open PRs with the label. The GitHub REST API paginates at 100 results per page, so loop through pages until empty.

## Skill: Find Open CK-Tile Core PRs

Goal: Find all open PRs in ROCm/rocm-libraries that modify files under `ck_tile/core/`.

## Step 1: Fetch open PRs and filter by file path
```bash
gh pr list --repo ROCm/rocm-libraries \
  --state open --limit 300 \
  --json number,title,author,createdAt \
  -q '[.[] | .number] | .[]' \
  | xargs -P6 -I{} sh -c '
      files=$(gh pr view {} --repo ROCm/rocm-libraries --json files \
        -q "[.files[].path | select(contains(\"ck_tile/core\"))] | join(\"\\n\")" 2>/dev/null)
      if [ -n "$files" ]; then
        meta=$(gh pr view {} --repo ROCm/rocm-libraries --json title,author,createdAt,url \
          -q "\"PR #{}  \\(.author.login)  \\(.createdAt[:10])  \\(.title)\"" 2>/dev/null)
        echo "$meta"
        echo "$files" | sed "s/^/  /"
        echo "---"
      fi
    '
```

## Step 2: Summarize each matched PR

For each PR found in Step 1, spawn a subagent with:
```
Summarize PR #<NUMBER> in ROCm/rocm-libraries focusing on ck_tile/core changes.

Run:
1. gh pr view <NUMBER> --repo ROCm/rocm-libraries --json title,body,url,author,number,files 2>&1
2. gh pr diff <NUMBER> --repo ROCm/rocm-libraries 2>&1 | head -800

Analyze and answer:
1. Which ck_tile/core files were modified? List them.
2. What category best describes this change?
   - Tile primitive (tile_window, tile_distribution, etc.)
   - Data type / numeric (fp8, bf16, type conversion, etc.)
   - Container / algorithm (array, sequence, reduce, etc.)
   - Config / arch dispatch (arch-specific guards, if constexpr, etc.)
   - Build / CMake
   - Other
3. One-sentence summary of what changed and why.

Format:
FILES: <comma-separated list of changed ck_tile/core files>
CATEGORY: <one of the above>
SUMMARY: <1 sentence>
```

Launch subagents in parallel (batch 6-8, `run_in_background: true`).

## Step 3: Compile results

Collect all subagent outputs into a table sorted by creation date (newest first):

| PR | Date | Author | Category | Summary |
|----|------|--------|----------|---------|
| #XXXX | YYYY-MM-DD | user | Category | Summary |
