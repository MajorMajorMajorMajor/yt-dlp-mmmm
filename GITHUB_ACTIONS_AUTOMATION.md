# GitHub Actions automation in this fork (`MajorMajorMajorMajor/yt-dlp-mmmm`)

This repo uses a small custom automation layer on top of upstream `yt-dlp` workflows.

## What it does

1. Poll upstream releases every 12 hours
2. Re-apply fork-specific commits (Threads fix + workflow customizations)
3. Push a generated `dist/main` branch
4. Build artifacts
5. Publish a GitHub Release with updater assets (`_update_spec`, `SHA2-256SUMS`, `yt-dlp`, etc.)

---

## Branch model

- `src/upstream`: mirror of upstream source commit/tag
- `src/threads`: custom extractor patch commits
- `src/automation`: custom CI/workflow commits
- `dist/main`: generated branch = upstream + cherry-picked custom commits

`dist/main` is the default branch and release source.

---

## Workflow map

### 1) `.github/workflows/sync.yml` — **Sync upstream + patches**

**Triggers**
- `schedule` (every 12 hours, cron `0 */12 * * *`)
- `workflow_dispatch`

**Main steps**
- resolve upstream source:
  - explicit `upstream_ref`, or
  - by `upstream_release_channel`:
    - `nightly`: `yt-dlp/yt-dlp-nightly-builds` latest release tag + target commit
    - `stable`: `yt-dlp/yt-dlp` latest stable release tag
- set `src/upstream` to upstream SHA
- recreate temporary branch from upstream
- cherry-pick commits from:
  - `src/threads`
  - `src/automation`
- compare against remote branch heads
- force-push:
  - `src/upstream`
  - `dist/main` (if changed and enabled)
- explicitly trigger `auto-release.yml`
- on conflict/failure, open an issue

**Key inputs**
- `upstream_ref` (default empty; when set, overrides channel selection)
- `upstream_release_channel` (`nightly`/`stable`, default `nightly`)
- `update_release_branch` (default `true`)
- `force_push` (default `false`)

**Feature flags (repo variables)**
- `SYNC_ENABLED` (`false` disables workflow job)
- `AUTO_RELEASE_ENABLED` (`false` prevents `dist/main` push + release trigger)

---

### 2) `.github/workflows/auto-release.yml` — **Release from dist/main**

**Triggers**
- `workflow_dispatch` (typically invoked by `sync.yml`)

**Jobs**
1. `prepare`
   - resolve release tag as `<upstream_tag>-mmmm` (or explicit `version` override)
   - resolve numeric `build_version` for `build.yml`
   - pass selected update `release_channel` (default `nightly`) to build
2. `build`
   - calls reusable `build.yml`
3. `publish`
   - download artifacts
   - verify required files exist
   - recreate release if same tag already exists
   - `gh release create`
   - verify `/releases/latest` contains required assets

All `gh release` commands use `--repo "${GITHUB_REPOSITORY}"`.

---

### 3) `.github/workflows/build.yml` — **Build Artifacts**

Called by `auto-release.yml` via `workflow_call`.

Builds platform binaries and updater metadata, including:
- `_update_spec`
- `SHA2-256SUMS`
- `yt-dlp`

---

## End-to-end flow

1. `sync.yml` runs every 12 hours (or manually)
2. It generates and pushes updated `dist/main`
3. It triggers `auto-release.yml`
4. `auto-release.yml` builds + publishes release assets
5. Clients can query `releases/latest`

---

## Manual operations

### Run sync now

```bash
gh workflow run sync.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref dist/main
```

### Run sync without updating `dist/main`

```bash
gh workflow run sync.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref dist/main \
  -f update_release_branch=false
```

### Run auto-release manually with upstream version + channel

```bash
gh workflow run auto-release.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref dist/main \
  -f upstream_version=2026.03.29.233709 -f release_channel=nightly
```

---

## Notes

- `dist/main` is generated output and can be force-pushed.
- Keep `src/threads` and `src/automation` minimal for easier conflict handling.
