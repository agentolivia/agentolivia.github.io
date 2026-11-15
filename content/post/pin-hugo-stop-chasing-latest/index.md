---
title: 'Why I Stopped Auto-Updating Hugo (and My Theme)'
description: My fix for a Hugo version mismatch quietly created a new one. Here's why I pinned Hugo, Dart Sass, and Go in one file and made theme updates something I run on purpose.
date: 2026-11-15T00:00:00-08:00
draft: true
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

Back in April I wrote about [a Hugo version mismatch that broke my Tugboat preview builds]({{< ref "/post/hugo-version-mismatch-automated-theme-updates" >}}). My fix was to make the daily theme-update workflow smarter: build the site before committing, open a PR instead of pushing to `main`, and automatically sync the Hugo version in `.tugboat/config.yml` to whatever the latest release was.

Months later, I asked Claude Code to review one of those automated PRs. The review showed the fix had created a new version of the same problem. That led me to a bigger question: why was I chasing the latest Hugo at all?

## The PR that wasn't a theme update

PR #30 was titled "CI: Update theme." The whole diff was one line:

```diff
- curl -Ls https://github.com/gohugoio/hugo/releases/download/v0.163.3/hugo_extended_0.163.3_Linux-64bit.tar.gz | tar -C /usr/local/bin -zxf - hugo
+ curl -Ls https://github.com/gohugoio/hugo/releases/download/v0.170.0/hugo_extended_0.170.0_Linux-64bit.tar.gz | tar -C /usr/local/bin -zxf - hugo
```

No theme changes. `go.mod` and `go.sum` were untouched. It was a Hugo bump with a theme-update label on it.

The bigger problem was where Hugo _wasn't_ being bumped. My production deploy workflow, `.github/workflows/deploy.yml`, pins its own Hugo version, and the sync step never touched it:

| Where                                       | Hugo version             |
| ------------------------------------------- | ------------------------ |
| `.github/workflows/deploy.yml` (production) | `0.161.1`                |
| `.tugboat/config.yml` (PR previews)         | `0.163.3` → `0.170.0`    |
| `.github/workflows/update-theme.yml`        | whatever is latest today |

The sync step kept Tugboat in step with the update workflow, but neither of those is what deploys my site. Every automated PR pushed my previews further ahead of production. A preview could pass on Hugo 0.170.0 while the real deploy ran 0.161.1, which defeats the point of having previews.

The update workflow's build check had the same blind spot. It built with the latest Hugo and Dart Sass, so a green check told me the site worked on versions production wasn't using.

## Three copies of the same number

The root cause wasn't the sed command. The Hugo, Dart Sass, and Go versions were hardcoded separately in three files, with one regex trying to keep one of them current. Any setup like that drifts eventually.

I could have fixed it by teaching the workflow to rewrite `deploy.yml` too. Before doing that, I asked a different question.

## Does it make sense to keep bumping Hugo?

For a personal blog, not really.

Hugo only runs at build time. It reads my Markdown and writes static HTML. None of it runs on a server, and nothing it does is exposed to visitors. Staying on an older version carries almost no security risk. New Hugo releases mostly bring features I may never use, plus the occasional deprecation that breaks a template.

So auto-bumping cost me reviews, drift, and surprise breakage, and gave me very little back. Pinning a version I know works is the better deal.

## What about the theme?

The theme is the one catch. If I keep auto-updating the theme, eventually a theme release will require a newer Hugo than my pin. That's exactly what happened in April. So I asked what I'd lose by stopping theme updates too:

- **Bug fixes and small improvements.** Most won't matter to me, and if one does, I'll notice and can update then.
- **Security fixes in the theme's JavaScript.** This is the real one. Unlike Hugo, the theme ships JavaScript that runs in visitors' browsers: search, the dark-mode toggle, the image gallery. Serious problems in a static theme are rare, but they're possible. Watching the theme's GitHub releases covers it.
- **Upgrading later is a bigger jump.** If I skip versions for years, I could face renamed config options and template changes all at once. Small updates spread that work out.
- **New features don't arrive automatically.** That only costs me something if I want them.

Weighed against that: every automatic update can change how my site looks or behaves without me asking, and I have to review each one. The workflow had been opening a theme PR roughly once a month.

I chose the middle path. Pin everything, turn off the schedule, and run updates by hand once or twice a year, or when I see a release I want.

## The change

### One file for versions

I added `versions.env` at the repo root:

```shell
HUGO_VERSION=0.161.1
DART_SASS_VERSION=1.97.1
GO_VERSION=1.25.7
```

These are the versions production was already deploying, so I knew they worked. It's plain `KEY=VALUE` with no quotes, which means it works both as a shell script (`. versions.env`) and as a GitHub Actions environment file.

### Production reads it

In `deploy.yml`, the hardcoded versions are gone. One step after checkout loads the file into the job's environment:

```yaml
- name: Load pinned tool versions
  run: |
    grep -E '^[A-Z_]+=' versions.env >> "${GITHUB_ENV}"
```

### Tugboat reads it too

Each Tugboat command runs in its own shell, so each install line loads the file first. While I was there, I made the Hugo download match the GitHub workflows: same `linux-amd64` archive name, `curl -f` so a bad URL fails with a clear HTTP error instead of piping an error page into `tar`, and a checksum check.

```yaml
- . "${TUGBOAT_ROOT}/versions.env" && cd /tmp && curl -fsSLO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" && curl -fsSLo hugo_checksums.txt "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_checksums.txt" && sha256sum --check --ignore-missing hugo_checksums.txt && tar -C /usr/local/bin -xzf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" hugo && rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz" hugo_checksums.txt && hugo version
```

Previews and production now build with the same Hugo by construction. There's nothing left to sync.

### Theme updates are on demand

`update-theme.yml` no longer has a `schedule`. It only runs when I start it from the Actions tab, and it asks which Hugo and Dart Sass versions I want:

```yaml
on:
  workflow_dispatch:
    inputs:
      hugo_version:
        description: "Hugo version: 'latest', 'pinned' (keep current), or a number like 0.161.1"
        required: true
        default: latest
      dart_sass_version:
        description: "Dart Sass version: 'latest', 'pinned' (keep current), or a number like 1.97.1"
        required: true
        default: latest
```

When I run it, it:

1. Works out the versions and writes them into `versions.env`.
2. Installs exactly those versions.
3. Runs `hugo mod get -u` to update the theme.
4. Builds the site, so the check now tests the versions that will deploy after merge.
5. Opens one PR with `go.mod`, `go.sum`, and `versions.env`.

Choosing `pinned` for both gives me a theme-only update. If a new theme release needs a newer Hugo, the build fails, no PR opens, and I know it's time to bump on purpose.

## Takeaways

In April, my answer to drift was more automation: sync the versions, then validate. That helped, but automation that keeps copies of a value in sync is still copies, and copies drift. One source of truth removes the problem rather than managing it.

The other lesson is to ask whether something needs updating at all. "Always run the latest" sounds responsible, but for a build-time tool on a personal blog it mostly bought me churn. Now updates happen when I decide, with all the versions changing together, and a preview I can trust before I merge.

I'm still keeping a human in the loop. The loop just runs when I start it now.
