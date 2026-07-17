# DaoMai Router MiniOS — Update Channel

This repo is the **update channel** for both parts of a DaoMai Router MiniOS release —
it does not hold the OS source itself. See
[daomai-router-minios](https://github.com/conkiubebong/daomai-router-minios) for the actual
MiniOS source (that repo may eventually go private; this one stays public so routers in the
field can keep updating).

Each GitHub [Release](../../releases) here carries two independent update paths for the same
version, and a booted router can use either one depending on the endpoint it calls:

- **Agent/web zip** — in-place hot update, no reboot. `GET /api/update/check` and
  `POST /api/update/apply` on `daomai-agent` download the release's `.zip` asset and replace the
  `daomai-agent` binary + `web/` directory only. Because the OS runs `toram`, this does not
  survive a reboot back onto the original ISO — it's for same-boot hotfixes.
- **Full ISO** — re-flashes the boot disk with a new `daomaios_<version>.iso`, so the update
  survives reboot. `POST /api/update/iso/apply` on `daomai-agent` downloads the release's `.iso`
  asset, verifies it against the accompanying `.iso.sha256` asset, backs up
  `/mnt/daomai-persist/router.db`, `dd`s the ISO over the boot disk, recreates/remounts the
  persistence partition, restores `router.db`, then reboots.

## What's in this repo

- `CHANGELOG.md` — human-readable list of what changed in each version.
- `update.json` — machine-readable pointer to the latest version (mirrors the GitHub Release, kept
  for reference/tooling; `daomai-agent` itself reads the GitHub Releases API directly, not this
  file).
- Release **assets** (not committed here) — each GitHub Release carries:
  - `daomai-router-minios-update-v<version>.zip` — the agent binary + web UI update package.
  - `daomaios_<version>.iso` — the full bootable ISO for that version.
  - `daomaios_<version>.iso.sha256` — SHA256 checksum of the ISO, one asset `daomai-agent`
    verifies before writing anything to disk.

## Update package format

A release's `.zip` asset must contain, at its top level:

```text
daomai-agent      # linux/amd64 binary
web/
  index.html      # (and any other static web UI files)
```

No `VERSION` file is needed inside the zip — the version comes from the GitHub Release's tag name
(e.g. `v0.2.0`), which `daomai-agent` writes to `/etc/daomai/VERSION` after applying.

## ISO asset format

- Filename must match `daomaios_<version>.iso` (same naming rule as the main repo's build script).
- Must be accompanied by a `.iso.sha256` (or `.sha256`) asset containing the checksum, e.g. the
  output of `sha256sum daomaios_<version>.iso`. `daomai-agent` refuses to write the ISO to disk
  if this asset is missing or the checksum doesn't match.
