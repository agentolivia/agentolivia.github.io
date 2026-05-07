---
title: "Fixing error: GitHub Actions isn't allowed to open pull requests"
description: After setting up the update-theme workflow to open PRs instead of pushing directly to main, I immediately found it wasn't actually allowed to do that. Then it complained about Node.js 20 actions being deprecated. Here's how I fixed both.
date: 2026-05-06T00:00:00-07:00
draft: false
comments: false
build:
  list: always
categories:
  - Tutorials
tags:
  - Hugo
  - GitHub
  - GitHub Actions
  - CI/CD
  - DevOps
  - Claude Code
---

In [my last post](/post/hugo-version-mismatch-automated-theme-updates/), I updated the `update-theme` workflow to open a pull request instead of committing theme updates directly to `main`. Smart in theory. In practice, when the workflow needed to open a pull request to bump the `hugo` version, it failed. The workflow run also warned me about Node 20 actions being deprecated.

I used Claude Code to help diagnose and work through both issues. This post is my note-to-future-self on what needed to change and where.

## Problem 1: GitHub Actions isn't allowed to open pull requests

The workflow failed with this error from the GitHub REST API:

> GitHub Actions is not permitted to create or approve pull requests.

The `peter-evans/create-pull-request` action uses the built-in `GITHUB_TOKEN` to open PRs. The workflow already had the right permissions declared:

```yaml
permissions:
  contents: write
  pull-requests: write
```

But there are **two separate settings** in the repository that also need to be enabled, and both were off by default.

### Setting 1: Actions allowlist

Go to **Settings → Actions → General** and look at the "Actions permissions" section. If you're using the "Allow [org], and select non-[org], actions and reusable workflows" option, you need to explicitly list the third-party actions your workflows use in the allowlist field.

Here's what I added:

```txt
peaceiris/actions-hugo@*,
peter-evans/create-pull-request@*,
errata-ai/vale-action@*
```

The `@*` wildcard covers any version tag, so this doesn't need updating when action versions get bumped. GitHub's own actions (`actions/checkout`, `actions/setup-node`, etc.) are covered separately by checking **"Allow actions created by GitHub"**.

### Setting 2: Workflow permissions

Further down on the same **Settings → Actions → General** page is the "Workflow permissions" section. There's a checkbox labeled **"Allow GitHub Actions to create and approve pull requests"** that is unchecked by default. Check it.

This is a separate gate from the `permissions:` block in the workflow YAML. Even with `pull-requests: write` in the workflow, the `GITHUB_TOKEN` cannot open PRs unless this repo-level setting is also enabled.

After enabling both, the workflow ran successfully and opened its first PR.

## Problem 2: Node.js 20 deprecation warnings

With the workflow actually running, a new batch of warnings appeared:

> Node.js 20 actions are deprecated. The following actions are running on Node.js 20 and may not work as expected: actions/checkout@v4, peaceiris/actions-hugo@v3, peter-evans/create-pull-request@v7. Actions will be forced to run with Node.js 24 by default starting June 2nd, 2026.

Node.js 20 reached end-of-life in April 2026. GitHub Actions is forcing a cutover to Node.js 24 on June 2nd, and removing Node.js 20 from runners entirely on September 16th.

This affected every workflow in the repo, not just `update-theme`. Claude Code helped me audit all five workflows and figure out what could be upgraded versus what needed a workaround.

### Action version bumps

Several actions had new major versions available with Node.js 24 support.

Cross-workflow updates (applied to all five workflow files where used):

| Action                            | Before  | After  |
| --------------------------------- | ------- | ------ |
| `actions/checkout`                | v4 / v5 | **v6** |
| `actions/setup-node`              | v4      | **v6** |
| `peter-evans/create-pull-request` | v7      | **v8** |

`deploy.yml`-specific updates:

| Action                          | Before | After  |
| ------------------------------- | ------ | ------ |
| `actions/setup-go`              | v5     | **v6** |
| `actions/configure-pages`       | v5     | **v6** |
| `actions/cache`                 | v4     | **v5** |
| `actions/upload-pages-artifact` | v3     | **v5** |
| `actions/deploy-pages`          | v4     | **v5** |

### Actions that don't have a Node.js 24 release yet

One action was already at its latest major version with no Node.js 24-compatible release available:

- `errata-ai/vale-action@v2` (used in `vale.yml`)

For it, I added the opt-in environment variable to the affected job:

```yaml
env:
  FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: true
```

This tells the runner to use Node.js 24 for that action now, before GitHub forces it on June 2nd. If it breaks, that's a signal to find an alternative or wait for an upstream update.

`peaceiris/actions-hugo@v3` was in the same situation — no Node.js 24 release, same warning — but I replaced it entirely rather than applying the workaround. See below.

### Replacing peaceiris/actions-hugo

`peaceiris/actions-hugo@v3` is an unmaintained community wrapper that downloads Hugo from GitHub releases and puts it on PATH. Since it has no Node.js 24-native release and is just a thin wrapper around a direct download, the right move was to remove the dependency and do the download directly — the same pattern already used for Dart Sass in `deploy.yml`.

In `deploy.yml`, where the Hugo version is pinned in an environment variable:

```yaml
- name: Setup Hugo
  run: |
    mkdir -p "${HOME}/.local/bin"
    curl -sLo "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
      "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
    curl -sLo "hugo_checksums.txt" \
      "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_checksums.txt"
    sha256sum --check --ignore-missing "hugo_checksums.txt"
    tar -C "${HOME}/.local/bin" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" hugo
    rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" "hugo_checksums.txt"
    echo "${HOME}/.local/bin" >> "${GITHUB_PATH}"
```

In `update-theme.yml`, where the workflow always wants the latest Hugo:

```yaml
- name: Setup Hugo
  env:
    GH_TOKEN: ${{ github.token }}
  run: |
    HUGO_VERSION=$(gh release view --repo gohugoio/hugo --json tagName --jq '.tagName | ltrimstr("v")')
    mkdir -p "${HOME}/.local/bin"
    curl -sLo "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" \
      "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
    curl -sLo "hugo_checksums.txt" \
      "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_checksums.txt"
    sha256sum --check --ignore-missing "hugo_checksums.txt"
    tar -C "${HOME}/.local/bin" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" hugo
    rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" "hugo_checksums.txt"
    echo "${HOME}/.local/bin" >> "${GITHUB_PATH}"
```

The `sha256sum --check --ignore-missing` step verifies the tarball against Hugo's published checksums file before extracting. `--ignore-missing` is needed because the checksums file covers all platforms and editions; we only downloaded one of them. This is something `peaceiris/actions-hugo` never did, so the replacement is actually a supply chain improvement over what it replaced.

### Aligning the pinned Node.js version

While auditing the workflows, I noticed the pinned `node-version` values were inconsistent:

| Workflow       | Node version (before) |
| -------------- | --------------------- |
| `deploy.yml`   | `24.12.0`             |
| `lint.yml`     | `22`                  |
| `security.yml` | `20`                  |

Node.js 22 is now in Maintenance LTS (its Active LTS phase ended October 2025). Node.js 24 is the current Active LTS and what `deploy.yml` was already using. I updated `lint.yml` and `security.yml` to `'24'` so all workflows are consistent.

## The two things to remember

If you set up a GitHub Actions workflow that opens PRs and it fails with a permissions error, check both places:

1. **Settings → Actions → General → Actions permissions** — add third-party actions to the allowlist
2. **Settings → Actions → General → Workflow permissions** — enable "Allow GitHub Actions to create and approve pull requests"

The `permissions:` block in the workflow YAML is necessary but not sufficient on its own. You need the repo-level checkbox too. Future me: check there first.
