# lifestream-collector (public)

A GitHub Actions runner that **collects alternate stream URLs** for the
**78,150 titles** the upstream refused, and pushes them to the private
[`lifestream-collector-data`](https://github.com/hinduaichat/lifestream-collector-data)
repo.

## Status: ✅ v1.0 Deployed (2026-10-26)

- **Initial sweep: 100% complete** — all 78,150 missing titles processed
- **7,072 streams recovered** from alternate providers (bowler, titan, giant, super_barbarian)
- **71,078 titles confirmed unavailable** — no provider returned sources for them
- Scheduled cron disabled (initial sweep complete)
- Manual workflow_dispatch available for on-demand re-sweeps
- Optional `reset` input parameter to re-sweep from cursor=0

## What this repo does

Each workflow run:

1. **Clones** the private data repo (using a `PAT_TOKEN` secret — see below)
2. **Runs** `reprobe.mjs` against the next batch of missing titles
3. **Checkpoints progress every 50 titles** (atomic write — no corruption)
4. **Git-pushes progress to remote every 500 titles** — survives runner death
5. **Reports** progress as a workflow summary

The reprobe script checks 4 providers (bowler, titan, giant, super_barbarian) via the
`api.framextv.cc` endpoint for each missing title.

## Output schema (in the private repo)

```
movie/<tmdbId>.json              ← movie sources (collected via collect.mjs)
tv/<tmdbId>/S01.json             ← season 1, contains [E01, E02, …]
tv/<tmdbId>/S02.json             ← season 2
anime/<anilistId>.json           ← anime sources
data/state.json                  ← collect.mjs workflow cursor
data/reprobe-state.json          ← reprobe.mjs workflow cursor + recoveredIds
data/recovered-titles.json       ← flat array of all recovered title keys
```

## Trigger a run

The workflow supports **manual dispatch only** (cron disabled after sweep completion):

1. Go to
   [Actions → Reprobe all missing streams](https://github.com/hinduaichat/lifestream-collector/actions/workflows/collect.yml)
2. Click **Run workflow**
3. Set the **batch size** (default 70000 = all remaining titles)
4. (Optional) Check **reset** to re-sweep from cursor=0

### Re-enabling the cron

If new missing titles appear in the future, uncomment the `schedule:` block in
`.github/workflows/collect.yml`:

```yaml
on:
  workflow_dispatch:
    inputs: { ... }
  schedule:
    - cron: "0 */6 * * *"   # every 6 hours
```

## Set the `PAT_TOKEN` secret

The workflow needs a GitHub Personal Access Token with `repo` scope to
clone and push to the private `lifestream-collector-data` repo.

```bash
# Get your PAT from https://github.com/settings/tokens (classic, repo scope)
gh secret set PAT_TOKEN --repo hinduaichat/lifestream-collector
# paste the token when prompted
```

Or via the GitHub web UI:
<https://github.com/hinduaichat/lifestream-collector/settings/secrets/actions>

## Concurrency

Only one run at a time — the `concurrency` block enforces this. Concurrent
runs would corrupt `data/reprobe-state.json`. `cancel-in-progress: false`
ensures queued runs wait their turn.

## Reliability features

| Feature | Purpose |
|---|---|
| Atomic state writes (temp + rename) | Prevents JSON corruption if killed mid-write |
| Checkpoint every 50 titles | Limits max progress loss to 50 titles on crash |
| Git push every 500 titles | Progress durable across runner restarts |
| 30s per-chunk timeout | Prevents API hangs from blocking the sweep |
| SIGTERM/SIGINT signal handlers | Final state flush before runner shutdown |
| `if: always()` on commit step | Commits partial progress even if script crashes |

## Run logs

Every run's logs are visible at:
<https://github.com/hinduaichat/lifestream-collector/actions>

---

© 2026 hinduaichat. All rights reserved.
