---
title: "When Your Theme Auto-Updates But Your Build Environment Doesn't"
description: How a Hugo version mismatch broke my Tugboat preview build, and how I fixed the workflow to catch this automatically—and add a human review step—in the future.
date: 2026-04-29T00:00:00-07:00
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
  - Tugboat
  - CI/CD
  - DevOps
---

When I was poking around my [Tugboat Dashboard](https://dashboard.tugboatqa.com/), I noticed the base preview for the `main` branch on this repo had failed. The site was still working, and it wasn't until today that I needed to get pull request previews up-and-running again for this repo. The error was a bit cryptic at first glance, but it turned out to be a straightforward version mismatch with a less-than-straightforward paper trail.

I used Claude Code in VS Code to help me diagnose and fix the error, and we worked together to devise a plan to update the update-theme workflow and Tugboat to be more **resilient to automatic changes and to put me—the human—in the loop when there was a need**. Claude Code helped me write a detailed PR description and turn this incident in a blog post, which I edited and added to. Robot teamwork!

## The error

```shell
WARN  Module "github.com/CaiJimmy/hugo-theme-stack/v4" is not compatible
      with this Hugo version: Min 0.157.0 extended
ERROR error building site: ... at <reflect>: can't evaluate field
      IsImageResourceWithMeta in type interface {}
```

Two errors, one cause. The theme ([hugo-theme-stack](https://github.com/CaiJimmy/hugo-theme-stack)) had been auto-updated to a version that requires Hugo **0.157.0 extended** as a minimum. My Tugboat config was pinned to **Hugo 0.155.3**. The `IsImageResourceWithMeta` template error isn't a separate bug—it's what happens when the theme tries to use a field that doesn't exist in the older Hugo version.

## How did this happen?

This site's repo includes a GitHub Actions workflow, `.github/workflows/update-theme.yml`, that runs on a daily cron schedule:

```yaml
- name: Update theme
  run: hugo mod get -u

- name: Tidy go.mod, go.sum
  run: hugo mod tidy

- name: Commit changes
  uses: stefanzweifel/git-auto-commit-action@v5
  with:
    commit_message: 'CI: Update theme'
```

That workflow installs `hugo-version: 'latest'` and then runs `hugo mod get -u`, which pulls the latest version of the theme and commits the result directly to `main`. **No build validation, no review step.** When the theme bumped its minimum Hugo requirement, the workflow committed the update anyway, the `main` branch now had an incompatible theme version, and Tugboat's next preview build failed.

The version mismatch lived in two different config files that had no awareness of each other:

| File                                 | Hugo version                       |
| ------------------------------------ | ---------------------------------- |
| `.github/workflows/update-theme.yml` | `latest` (always current)          |
| `.tugboat/config.yml`                | `0.155.3` (manually pinned, stale) |

## The immediate fix

The quickest fix was updating `.tugboat/config.yml` to install a Hugo version that satisfies the theme's requirements. Hugo 0.161.1 is the current latest release, so I bumped the download URL there:

```yaml
# Before
- curl -Ls https://github.com/gohugoio/hugo/releases/download/v0.155.3/hugo_extended_0.155.3_Linux-64bit.tar.gz | tar -C /usr/local/bin -zxf - hugo

# After
- curl -Ls https://github.com/gohugoio/hugo/releases/download/v0.161.1/hugo_extended_0.161.1_Linux-64bit.tar.gz | tar -C /usr/local/bin -zxf - hugo
```

That unblocks Tugboat, but it doesn't prevent the same thing from happening the next time the theme bumps its minimum Hugo requirement. I needed to rethink the workflow.

## Making it future-proof

The core problem is that the workflow had no feedback loop: it updated the theme, assumed everything was fine, and committed. I made three changes to address that.

### 1. Auto-sync the Tugboat Hugo version (automated)

The workflow already installs the latest Hugo to run `hugo mod get -u`. After updating the theme, I extract that version number and rewrite the download URL in `.tugboat/config.yml` to match:

```yaml
- name: Sync Hugo version in Tugboat config
  run: |
    HUGO_VERSION=$(hugo version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
    sed -i -E "s|/download/v[0-9.]+/hugo_extended_[0-9.]+_Linux-64bit|/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit|g" .tugboat/config.yml
```

<details>
<summary>What does that command do?</summary>

The two lines work together to extract the installed Hugo version and rewrite the download URL in `.tugboat/config.yml`.

### Line 1 — extract the version number

`hugo version` outputs something like `hugo v0.161.1+extended linux/amd64 ...`. The pipeline extracts just `0.161.1`:

- `grep -oE '[0-9]+\.[0-9]+\.[0-9]+'` — `-o` prints only the matching text (not the whole line); `-E` enables extended regex; the pattern matches any `digits.digits.digits` sequence
- `| head -1` — takes the first match, in case the output ever contains more than one version-like string

The result is stored in `HUGO_VERSION`.

### Line 2 — rewrite the URL in `.tugboat/config.yml`

`sed -i -E "s|old|new|g"` is a find-and-replace on the file:

- `-i` — edit in place (modifies the file directly, no output)
- `-E` — enables extended regex, needed for `+` to mean "one or more"
- `s|...|...|g` — the substitute command, using `|` as the delimiter instead of the usual `/` because the pattern itself contains forward slashes, which would otherwise need escaping

The pattern matches the version-specific portion of the Hugo download URL—`[0-9.]+` matches any version number like `0.155.3` or `0.161.1`. The replacement plugs `$HUGO_VERSION` into both spots where the version number appears. The `g` flag replaces all occurrences, though there's only one matching line in the file.

</details>

Now the workflow keeps both environments in lock-step automatically. Whatever Hugo version the workflow installs and validates against is the same version Tugboat gets in the same commit.

### 2. Build validation (automated)

After updating the theme and syncing the Tugboat config, the workflow now tries to actually build the site:

```yaml
- name: Build site
  run: hugo --gc --minify
```

If the build fails—bad template, incompatible theme change, anything—the workflow stops here. Nothing gets committed. **This is the automated gate: it catches functional breakage before it touches the repo.**

### 3. Open a PR instead of pushing directly to `main` (human in the loop)

Even if the build succeeds, `hugo mod get -u` is pulling in external code and committing it automatically. A theme update that builds cleanly could still introduce changes worth reviewing—a new JavaScript dependency, a layout change, a modified partial. The original workflow gave no opportunity to see any of that.

I replaced the auto-commit action with `peter-evans/create-pull-request`, which opens a PR on a branch (`automated/update-theme`) instead of pushing to `main`:

```yaml
- name: Create Pull Request
  uses: peter-evans/create-pull-request@v7
  with:
    commit-message: 'CI: Update theme'
    title: 'CI: Update theme'
    body: |
      Automated theme update via `hugo mod get -u`. Please review changes before merging.
    branch: automated/update-theme
    delete-branch: true
    add-paths: |
      go.mod
      go.sum
      .tugboat/config.yml
```

A few details worth calling out:

- **`add-paths`** scopes the PR to only the three files that should change. The build step generates output in `public/`, which we don't want to commit.
- **`delete-branch: true`** cleans up the branch after the PR merges.
- If the theme is already up to date, there are no changes to the specified paths and no PR is opened. No noise on quiet days.
- `pull-requests: write` was added to the job permissions to allow the action to open PRs.

## The updated workflow

Here's the full workflow after these changes:

```yaml
name: Update theme

on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:

jobs:
  update-theme:
    runs-on: ubuntu-latest

    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: 'latest'
          extended: true

      - name: Update theme
        run: hugo mod get -u

      - name: Tidy go.mod, go.sum
        run: hugo mod tidy

      - name: Sync Hugo version in Tugboat config
        run: |
          HUGO_VERSION=$(hugo version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | head -1)
          sed -i -E "s|/download/v[0-9.]+/hugo_extended_[0-9.]+_Linux-64bit|/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit|g" .tugboat/config.yml

      - name: Build site
        run: hugo --gc --minify

      - name: Create Pull Request
        uses: peter-evans/create-pull-request@v7
        with:
          commit-message: 'CI: Update theme'
          title: 'CI: Update theme'
          body: |
            Automated theme update via `hugo mod get -u`. Please review changes before merging.
          branch: automated/update-theme
          delete-branch: true
          add-paths: |
            go.mod
            go.sum
            .tugboat/config.yml
```

## Takeaways

Automation that commits directly to your main branch with no validation is convenient right up until it isn't. The combination that works better here is:

- **Automated gates** to catch objective failures (the build doesn't compile, the version numbers are out of sync)
- **Human review** for everything that passes the automated gate but still involves external code landing in your repo

Neither one alone is enough. The build gate would have caught the broken template error in this case—but a future theme update that builds cleanly and introduces something undesirable would still slip through without the PR step. And the PR step alone, without the build gate, would still surface broken builds in review instead of catching them before the PR is even opened.

Layers are good.
