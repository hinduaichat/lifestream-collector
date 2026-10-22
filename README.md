# lifestream-collector (public)

A GitHub Actions runner that collects alternate stream URLs for the
**78,150 titles** the upstream refused, and pushes them to the private
[`lifestream-collector-data`](https://github.com/hinduaichat/lifestream-collector-data)
repo.

## What this repo does

Every workflow run:

1. **Clones** the private data repo (using a `PAT_TOKEN` secret — see below)
2. **Runs** `collect.mjs` against the next batch of missing titles
3. **Commits and pushes** the results back to the private repo
4. **Reports** progress as a workflow summary

The collector probes 5 public streaming aggregators for each missing title:

| Source | Endpoint |
|---|---|
| VidSrc.to | `https://vidsrc.to/v1/<type>/<id>[/<s>/<e>]` |
| VidSrc.xyz | `https://vidsrc.xyz/embed/<type>/<id>[/<s>/<e>]` |
| MultiEmbed | `https://multiembed.mov/?video_id=<id>&tmdb=1[&s=<s>&e=<e>]` |
| 2Embed.cc | `https://www.2embed.cc/embed/<id>[/<s>/<e>]` |
| Embed.su | `https://embed.su/embed/<type>/<id>[/<s>/<e>]` |

For each title, the first valid `m3u8` or `mp4` URL from each source is
kept (up to 3 sources). For TV shows, the first episode of each season
(up to 5) is probed.

## Output schema (in the private repo)

```
movie/<tmdbId>.json              ← movie sources
tv/<tmdbId>/S01.json             ← season 1, contains [E01, E02, …]
tv/<tmdbId>/S02.json             ← season 2
anime/<anilistId>.json           ← anime sources
data/state.json                  ← workflow cursor
```

## Trigger a run

This repo has TWO triggers:

1. **Manual** — go to
   [Actions → Collect missing streams](https://github.com/hinduaichat/lifestream-collector/actions/workflows/collect.yml)
   and click **Run workflow**. You can set the batch size (default 200).
2. **Scheduled** — every 6 hours, automatically.

## Set the `PAT_TOKEN` secret

The workflow needs a GitHub Personal Access Token with `repo` scope to
clone and push to the private `lifestream-collector-data` repo.

To set it:

```bash
# Get your PAT from https://github.com/settings/tokens (classic, repo scope)
gh secret set PAT_TOKEN --repo hinduaichat/lifestream-collector
# then paste the token when prompted
```

Or via the GitHub web UI:
<https://github.com/hinduaichat/lifestream-collector/settings/secrets/actions>

## Concurrency

Only one run at a time — the `concurrency` block enforces this. Concurrent
runs would corrupt `data/state.json`.

## Run logs

Every run's logs (including which titles were collected, which failed,
and why) are visible at:

<https://github.com/hinduaichat/lifestream-collector/actions>

---

© 2026 hinduaichat. All rights reserved.
