# GitHub Actions automation in this fork (`MajorMajorMajorMajor/yt-dlp-mmmm`)

This repo uses a small custom automation layer on top of upstream `yt-dlp` workflows.

## What it does

1. Poll upstream `yt-dlp/yt-dlp` releases every 30 minutes
2. Re-apply fork-specific commits (Threads fix + workflow customizations)
3. Push a generated `release/main` branch
4. Build artifacts
5. Publish a GitHub Release with updater assets (`_update_spec`, `SHA2-256SUMS`, `yt-dlp`, etc.)

---

## Branch model

- `upstream-sync`: mirror of upstream `master`
- `feature/threads-fix`: custom extractor patch commits
- `automation/workflows`: custom CI/workflow commits
- `release/main`: generated branch = upstream + cherry-picked custom commits

`release/main` is the default branch and the release source.

---

## Workflow map

### 1) `.github/workflows/sync.yml` — **Sync upstream + threads patch**

**Triggers**
- `schedule` (every 30 min, cron `*/30 * * * *`)
- `workflow_dispatch`

**Main steps**
- resolve upstream source:
  - explicit `upstream_ref`, or
  - latest upstream release by `upstream_release_channel`:
    - `nightly`: `yt-dlp/yt-dlp-nightly-builds` latest release tag + target commit
    - `stable`: `yt-dlp/yt-dlp` latest stable release tag
- set `upstream-sync` to upstream SHA
- recreate temporary branch from upstream
- cherry-pick commits from:
  - `feature/threads-fix`
  - `automation/workflows`
- compare against remote branch heads
- force-push:
  - `upstream-sync`
  - `release/main` (if changed and enabled)
- explicitly trigger `auto-release.yml`
- on conflict/failure, open an issue

**Key inputs**
- `upstream_ref` (default empty; when set, overrides channel selection)
- `upstream_release_channel` (`nightly`/`stable`, default `nightly`)
- `update_release_branch` (default `true`)
- `force_push` (default `false`)

**Feature flags (repo variables)**
- `SYNC_ENABLED` (`false` disables workflow job)
- `AUTO_RELEASE_ENABLED` (`false` prevents release/main push + trigger)

---

### 2) `.github/workflows/auto-release.yml` — **Release from release/main**

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

**Important implementation detail**
All `gh release` commands use `--repo "${GITHUB_REPOSITORY}"` so they work reliably in non-checkout contexts.

---

### 3) `.github/workflows/build.yml` — **Build Artifacts** (reusable upstream workflow)

Called by `auto-release.yml` via `workflow_call`.

Builds platform binaries and updater metadata, including:
- `_update_spec`
- `SHA2-256SUMS`
- `yt-dlp` (plus many platform-specific outputs)

---

## End-to-end flow

1. `sync.yml` runs every 30 minutes (or manually)
2. It generates and pushes updated `release/main`
3. It triggers `auto-release.yml`
4. `auto-release.yml` builds + publishes release assets
5. Clients can query `releases/latest`

---

## Permissions required

- In workflow files:
  - `sync.yml`: `contents: write`, `issues: write`, `actions: write`
  - `auto-release.yml`: `contents: write`
- Repo settings must allow Actions and write operations for `GITHUB_TOKEN`.

---

## Manual operations

### Run sync now

```bash
gh workflow run sync.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref release/main
```

### Run sync without updating `release/main`

```bash
gh workflow run sync.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref release/main \
  -f update_release_branch=false
```

### Run auto-release manually with upstream version + channel

```bash
gh workflow run auto-release.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref release/main \
  -f upstream_version=2026.03.29.233709 -f release_channel=nightly
```

### Run auto-release manually with explicit release tag override

```bash
gh workflow run auto-release.yml -R MajorMajorMajorMajor/yt-dlp-mmmm --ref release/main \
  -f version=2026.03.17-mmmm
```

---

## Troubleshooting

### No releases created
- Check `Release from release/main` workflow runs
- Confirm `AUTO_RELEASE_ENABLED` is not `false`
- Confirm `sync.yml` reached:
  - push `release/main`
  - trigger `auto-release.yml`

### `releases/latest` missing assets
- Inspect `publish` step in `auto-release.yml`
- Ensure required files exist in downloaded artifacts:
  - `_update_spec`
  - `SHA2-256SUMS`
  - `yt-dlp`

### Cherry-pick conflict during sync
- Workflow opens an issue with upstream SHA
- Rebase/fix commits in `feature/threads-fix` or `automation/workflows`
- rerun `sync.yml`

---

## Notes

- `release/main` is generated output and can be force-pushed.
- Keep patch branches small and focused for easier conflict resolution.
- Upstream test/release workflows still exist; custom automation is centered on `sync.yml` + `auto-release.yml`.
