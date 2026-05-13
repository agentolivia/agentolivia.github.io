---
title: 'Hardening GitHub Actions workflows against supply chain attacks'
description:
  A walkthrough of the specific changes I made to harden my workflows against
  supply chain attacks via third-party GitHub Actions and npm, plus a prompt
  you can use to run the same review on your own repo.
date: 2026-05-13T00:00:00-07:00
draft: true
comments: false
build:
  list: always
categories:
  - Tutorials
tags:
  - GitHub
  - GitHub Actions
  - CI/CD
  - DevOps
  - Security
  - Claude Code
---

I asked Claude Code to review the GitHub Actions workflows in this repo for supply chain attack risk. This post covers what it found, what we changed, and a prompt you can reuse on your own workflows.

## The threat model

A supply chain attack via GitHub Actions works like this: a third-party action you depend on (say, `some-org/some-action@v2`) gets its tag force-pushed with malicious code. On your next workflow run, that code executes in your CI environment. It has access to your secrets and token permissions.

The same risk applies to npm. If a package in your `node_modules` tree is compromised, its postinstall script runs during `npm ci`. It can read any credentials in the job environment.

Neither is theoretical. Both attack types have hit real CI pipelines.

## What I changed

### 1. Pin third-party actions to commit SHAs

GitHub's version tags (e.g., `@v2`, `@v8`) are mutable. A tag can be moved to point to a different commit without notice. Pinning to a commit SHA means the code you reviewed is the code that runs, forever.

<details>
<summary>Immutable vs mutable</summary>

Git tags are just labels. The `v8` tag is a pointer, and whoever controls the repo can move it to a different commit. Same name, different code. That's what mutable means here.

A commit SHA is different. Git derives it from the commit's content, so it can only refer to that one thing. You can't redirect it.

That's what makes SHA pinning an actual security control. Once you've reviewed what's at a given SHA, nobody can swap something else in without you noticing.

</details>

I was using two third-party actions:

```yaml
# Before
uses: peter-evans/create-pull-request@v8
uses: errata-ai/vale-action@v2

# After
uses: peter-evans/create-pull-request@5f6978faf089d4d20b00c7766989d076bb2fc7f1 # v8
uses: errata-ai/vale-action@d89dee975228ae261d22c15adcd03578634d429c # v2
```

The `# v8` comment keeps it human-readable. To get the SHA for a tag:

```bash
gh api repos/peter-evans/create-pull-request/git/ref/tags/v8 --jq '.object.sha'
```

GitHub's own first-party `actions/*` are lower risk, but if your threat model requires it, the same pinning applies.

### 2. Add explicit `permissions` blocks

Without an explicit `permissions` block, a workflow job inherits the repository's default token permissions, which can be `read/write` depending on repo settings. Declaring minimum permissions limits what an attacker can do with a compromised token.

Three of my workflow files had no `permissions` block at all:

```yaml
# Added to lint.yml, security.yml, vale.yml
permissions:
  contents: read
```

The `deploy.yml` and `update-theme.yml` workflows were already scoped correctly. `deploy.yml` needs `pages: write` and `id-token: write` for GitHub Pages deployment. `update-theme.yml` needs `contents: write` and `pull-requests: write` to create automated PRs.

### 3. Set `persist-credentials: false` on checkouts that don't need to push

By default, `actions/checkout` writes the `GITHUB_TOKEN` into `.git/config` where it stays accessible to every subsequent step in the job. That includes `npm ci`, which runs third-party postinstall scripts.

For jobs that check out code but never push (linting, security scanning, prose review), there's no reason for the credential to persist:

```yaml
- uses: actions/checkout@v6
  with:
    persist-credentials: false
```

I applied this to `lint.yml`, `security.yml`, and `vale.yml`. The `deploy.yml` and `update-theme.yml` workflows keep the default because they run git operations that need authentication.

### 4. Add `timeout-minutes` to every job

Jobs without a timeout run for up to 6 hours by default. That's a long window of live token exposure if something goes wrong. Tight timeouts also limit the damage from a hung or hijacked job.

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    timeout-minutes: 5
```

Each of these jobs completes in seconds, with the build and deploy pipeline taking under two minutes. I set all jobs to 5 minutes. That's a reasonable buffer given actual runtimes, and a lot better than leaving the 6-hour default in place.

## The prompt

Here's the prompt I used. It works well with Claude Code in a repo with workflow files present:

---

> Review the GitHub Actions workflows in `.github/workflows/` for supply chain attack risk. For each workflow file, check:
>
> 1. **Action pinning** — Are any third-party actions (non-`actions/*`) pinned to a mutable version tag instead of a commit SHA? If so, fetch the current SHA for each tag using `gh api repos/<owner>/<repo>/git/ref/tags/<tag>` and pin them. Add the version tag as a comment for readability.
> 2. **Permissions** — Does every job have an explicit `permissions` block declaring the minimum required scopes? If not, add one. Jobs that only read code should use `permissions: contents: read`. Note which jobs legitimately need write scopes and why.
> 3. **`persist-credentials`** — Does `actions/checkout` default to persisting the `GITHUB_TOKEN` into `.git/config`? For any job that does not need to `git push`, add `persist-credentials: false` to the checkout step. This prevents the token from being readable by npm postinstall scripts or other downstream steps.
> 4. **Timeouts** — Does every job have a `timeout-minutes` value set? If not, add one. Base the value on a realistic upper bound for that job's expected runtime.
>
> For each finding, explain the risk and make the change. Do not pin GitHub's own first-party `actions/*` actions unless I ask — focus on third-party dependencies.

---

Run this against a repo with a clean working tree. The diff lands in one place, which makes it easier to spot anything unexpected before you commit.

## What this doesn't cover

This review focuses on the workflow files themselves. It doesn't cover:

- **Repository secrets** — if you have a personal access token (PAT) with broad org-wide scope stored as a repo secret, that's a higher-impact risk than anything in the workflow YAML. Audit your repo and org secrets separately.
- **`pull_request_target`** — workflows using this trigger run with write permissions even for PRs from forks, which can give fork contributors unintended write access to your repo. None of my workflows use it, but it's worth checking yours.
- **Dependabot for Actions** — automatically opens PRs to bump action versions. Pairing it with SHA pinning requires extra config but keeps pinned SHAs from going stale.
