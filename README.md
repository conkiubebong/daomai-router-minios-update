# DaoMai Router MiniOS — Update Channel

This repo is only the **update channel** for `daomai-agent` (the DaoMai Router MiniOS agent) —
it does not hold the OS image itself. See
[daomai-router-minios](https://github.com/conkiubebong/daomai-router-minios) for the actual
MiniOS ISO and source.

An already-booted router checks this repo's [Releases](../../releases) for a newer version and,
if found, downloads the release's `.zip` asset and applies it in place (binary + web UI only —
`/etc/daomai/config.yaml` is never touched) via `GET /api/update/check` and
`POST /api/update/apply` on `daomai-agent` itself. No need to re-flash the whole ISO just to pick
up a newer agent build.

## What's in this repo

- `CHANGELOG.md` — human-readable list of what changed in each version.
- `update.json` — machine-readable pointer to the latest version (mirrors the GitHub Release, kept
  for reference/tooling; `daomai-agent` itself reads the GitHub Releases API directly, not this
  file).
- Release **assets** (not committed here) — each GitHub Release carries the actual `.zip` update
  package (`daomai-agent` binary + `web/` directory).

## Update package format

A release's `.zip` asset must contain, at its top level:

```text
daomai-agent      # linux/amd64 binary
web/
  index.html      # (and any other static web UI files)
```

No `VERSION` file is needed inside the zip — the version comes from the GitHub Release's tag name
(e.g. `v0.2.0`), which `daomai-agent` writes to `/etc/daomai/VERSION` after applying.
