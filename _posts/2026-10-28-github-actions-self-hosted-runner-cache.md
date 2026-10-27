---
layout: post
title: "Out of GitHub Actions Minutes? How to Move CI to a Self-Hosted Runner Without Slowing It Down"
slug: github-actions-self-hosted-runner-cache
permalink: /blog/github-actions-self-hosted-runner-cache/
date: 2026-10-28
image: /assets/images/blog/github-actions-self-hosted-runner-cache/cache-before-after.png
description: "Out of GitHub Actions minutes, I moved CI to a self-hosted runner and it got 5x slower. Caching defaults built for GitHub's throwaway runners work backwards on your own machine. The fixes, measured."
keywords:
  - github actions minutes ran out
  - github actions self-hosted runner
  - self-hosted runner slow
  - setup-uv enable-cache self-hosted
  - github actions cache pull request scope
  - docker buildx cache-to type=gha pull request
  - docker build cached but slow
  - setup-buildx-action keep-state
  - runs-on repository variable
tags: [github-actions, ci, self-hosted-runner, uv, docker, python]
---

I ran out of GitHub Actions minutes five days before the reset, so I moved CI onto my own laptop. It got five times slower. The fix was turning a cache off.

The cache was downloading 1.24 GB of files that were already on the laptop's disk, on every run. That's not a bug: the setting is exactly right on GitHub's hosted runners, where every job starts on a blank machine, and backwards on a self-hosted runner, which keeps its disk between jobs. It wasn't the only one: I found three caching defaults like it in a day.

GitHub Free's [2,000 Actions minutes a month](https://docs.github.com/en/billing/reference/product-usage-included) are shared across every private repo on the account, and three of my apps draw on them. Paying the [overage](https://docs.github.com/en/billing/concepts/product-billing/github-actions) would have cost about $1.50 for those five days, which was a perfectly good answer. I chose a self-hosted runner because three apps on 2,000 minutes will hit the limit every month, and self-hosted minutes are free, for now: GitHub announced a $0.002 per-minute charge for them from March 2026, then [postponed it](https://www.theregister.com/2025/12/17/github_charge_dev_own_hardware/) without cancelling it. The repository is private, so the snippets below are the real ones with the names changed.

## Cache one: a 1.24 GB copy of my own disk

The app is an unreleased FastAPI service on Postgres. Its `check` job runs ruff, Alembic migrations against an empty database, `alembic check` and pytest against a Postgres 16 service container, and a separate `image` job builds the Docker image. On GitHub's runners, `check` took one to two minutes. On the laptop it took about ten. The job log didn't leave much room for argument: of roughly 590 seconds, 534 went to a single step, restoring the uv cache.

![The check job's steps on the self-hosted runner: setup-python-env takes 8m 54s of a 9m 47s job with GitHub's cache restored, and 2s with enable-cache left at its default](/assets/images/blog/github-actions-self-hosted-runner-cache/cache-before-after.png)

*Left: the run with the cache restored. Right: a later run after the fix; the timings quoted below come from the first run after the change.*

All my Python jobs go through a small composite action that installs uv with `astral-sh/setup-uv`, and it forced `enable-cache: true`. On GitHub-hosted runners that's the right call. Every job starts on a blank machine, and restoring uv's cache from GitHub's cache storage beats downloading every package from PyPI again.

On my laptop, uv's cache is `~/.cache/uv`, a directory that was already on the disk with 4.5 GB in it. The first self-hosted run found no cache entry for its key, so at the end of the job it saved a 1.24 GB slice of that directory to GitHub. [Caches written from self-hosted runners still live in GitHub's cloud storage](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching), so every run after that downloaded 1.24 GB over a home connection, back onto the machine it had come from.

The fix deletes one line:

```diff
     - uses: astral-sh/setup-uv@v10.3.2
       with:
         version: "0.9.27"
-        enable-cache: true
         cache-dependency-glob: uv.lock
```

The default is `enable-cache: auto`, and [setup-uv](https://github.com/astral-sh/setup-uv) already does the right thing with it: it caches on GitHub-hosted runners and skips the cache on self-hosted ones. In v10.3.2 that's a check on `RUNNER_ENVIRONMENT`, which is only `github-hosted` on GitHub's machines. I had overridden a default that knew better. Then I removed the stale entry with `gh cache delete <id>`, so nothing could restore it.

The next run's `check` took about 50 seconds, with 3 of them on Python setup. The work the job exists to do is small: ruff in under a second, migrations in 2, `alembic check` in 1, pytest in 29. Everything else had been overhead. That's faster than `check` ever was on GitHub, because the dependency cache is now genuinely local.

## Cache two: Docker layers nobody is allowed to read

The Docker build used BuildKit's GitHub Actions cache on every run, pull requests included, with `cache-to: type=gha,mode=max`. The cache list had entries scoped to `refs/pull/42`, `43`, `44` and `45`.

The same docs' rules for who can read a cache make those entries close to useless. A cache written by a pull request run is scoped to that PR's merge ref. Re-runs of the same PR can restore it. The base branch can't, and neither can any other PR.

So every PR uploaded layers that nothing else would read. They still cost upload time, now over a home connection, and space in the repository's cache, which is limited to 10 GB by default. When it fills, GitHub evicts the oldest entries first, and those can be `main`'s, the only entries every PR is allowed to use.

The fix is to read the cache everywhere and write it only when pushing, which only CD on `main` does:

{% raw %}
```yaml
cache-from: type=gha
# Only CD writes the cache: entries a PR writes can't be read by other PRs.
cache-to: ${{ inputs.push == 'true' && 'type=gha,mode=max' || '' }}
```
{% endraw %}

The pull request that made this change wasn't any faster for it: its image job took 2:42, against 2.6 minutes before. A PR that doesn't change dependencies has little new to upload. The win shows up on PRs that touch dependencies or the Dockerfile, and in a cache that only holds entries something will read.

## Cache three: every layer CACHED, seven minutes anyway

Merging to `main` runs both jobs again, then pushes the image to GHCR and deploys it to Dokku. The first deploy from the laptop turned up the strangest of the three: Docker marked every layer of the build `CACHED`, and the build still took over seven minutes. The whole deploy took 588 seconds, against about two minutes on GitHub.

| Step | Laptop | GitHub |
|---|---|---|
| Image build | 434 s | 13 s |
| Dokku deploy | 117 s | 103 s |

The Dokku step is the server pulling the image and restarting, so the laptop barely touches it, and the push to GHCR took 4.7 seconds because the layers were already in the registry. The build was the problem.

`CACHED` means BuildKit didn't have to rebuild the layer. It doesn't mean the layer was on the machine. With `type=gha`, the cache is in GitHub's storage, and to produce the final image the builder still downloaded about 200 MB of layers (138 MB of it the dependency layer) at about 0.5 MB/s. That was 405 of the 434 seconds. On GitHub's runners the cache sits next door, so the same download takes seconds.

![The deploy's build-image log: the dependency layer is marked CACHED at 20:33:35, then its 137.96 MB of blobs download slowly, and the next build step starts at 20:40:20](/assets/images/blog/github-actions-self-hosted-runner-cache/cached-but-slow.png)

*Every layer `CACHED` by 20:33:35. The dependency layer's 137.96 MB then downloads from GitHub's cache, and the next step doesn't start until 20:40:20.*

Under that is a third throwaway-machine default. `docker/setup-buildx-action` creates a fresh, empty builder for every job and removes it at the end, so even on a machine that keeps its disk, BuildKit's local layers vanish after every run. The action has an input for exactly this case, `keep-state`, which its docs describe as "only useful on persistent self-hosted runners":

{% raw %}
```yaml
- uses: docker/setup-buildx-action@v4
  with:
    name: ci
    keep-state: ${{ runner.environment == 'self-hosted' }}
```
{% endraw %}

The state is kept in a Docker volume named after the builder, `buildx_buildkit_ci0_state`, which is why the builder gets a fixed name. The condition keeps GitHub's runners on the old behaviour.

The first build after the change took 162 seconds and left that volume behind. The next one took **19 seconds**, with every layer `CACHED` and no layer downloads in the log at all.

All three fixes are the same idea. A remote cache earns its keep when the machine is thrown away after every job. On a machine that keeps its disk, it's a slow round trip of state you already have. If you're moving to a self-hosted runner, check every cache setting in your workflows and what each action does when it isn't on a GitHub-hosted runner.

## Switching runners with one variable

I didn't want moving CI to be a one-way door. Every job's `runs-on` reads a repository variable, with GitHub's runner as the fallback:

{% raw %}
```yaml
runs-on: ${{ vars.RUNNER || 'ubuntu-latest' }}
```
{% endraw %}

`gh variable set RUNNER --body self-hosted` sends jobs to my machine, and `gh variable delete RUNNER` sends them back to GitHub, with no commit either way. The variable can also hold a custom runner label, such as `spare`, to target one particular machine later. It doesn't fail over by itself: if the variable says `self-hosted` and the laptop is closed, jobs sit in the queue, and GitHub fails them after 24 hours.

## Port 5432 is taken on a real machine

CI starts Postgres as a service container mapped to `5432:5432`. GitHub's runners are fresh machines, so 5432 is always free. My laptop runs its own Postgres, which already holds it. The fix is to let Docker pick a free host port and read it back. The catch is that `job.services` isn't available in a job-level `env:` block, only inside steps, so a step writes the connection string to `$GITHUB_ENV`:

{% raw %}
```yaml
services:
  db:
    image: postgres:16
    ports:
      - 5432          # was 5432:5432
steps:
  - uses: actions/checkout@v7
  - name: Point at the service's random host port
    run: echo "DATABASE_URL=postgresql://app:app@localhost:${{ job.services.db.ports['5432'] }}/app" >> "$GITHUB_ENV"
```
{% endraw %}

## One runner, one queue

Merging two pull requests close together showed the last difference. GitHub runs jobs side by side; one runner takes them in order.

The CD workflow checks that `main` still points at the commit it's about to deploy. The first merge's deploy waited 224 seconds for the runner, saw that `main` had moved on, and skipped itself in 30 seconds. The second merge's deploy did the real work. The guard did its job, but the queue still cost time: the second merge's `check` waited 255 seconds behind other jobs, and it took about 18 minutes from the first merge to the last deploy finishing. On GitHub, each merge took about 4.

The rest of the trade-off:

- **Downloads are slower.** GitHub's runners sit in data centres. Anything a job pulls from the network comes in over my connection.
- **It only works while the laptop is on.** The service file the runner ships with also doesn't restart it after a crash.
- **Never attach a self-hosted runner to a public repository.** Anyone who opens a pull request can run code on your machine.
- **The runner holds deploy secrets while a job runs.** For a private repository with one maintainer, that's a trust boundary I'm comfortable with. With other contributors, it wouldn't be.
- **A personal account can't share one runner across repositories.** Each repository registers its own; sharing one needs an organisation, which is free.

`check` now takes about 50 seconds, faster than it ever did on GitHub, and a warm image build takes 19. The next problem is that CI stops whenever I close the laptop, and moving the runner to a spare machine is a post of its own.
