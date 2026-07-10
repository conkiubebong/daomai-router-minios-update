# Changelog

## v0.4.0 — 2026-07-10

- Mobile self-service page overhaul: own vi/en i18n dictionary with phone-local language
  detection, delimiter combobox matching the desktop's exact labels, Save/Delete-proxy buttons
  moved to their own section above Proxy, page reordered to buttons → Proxy → Other options →
  device info, idle status shows the local IP instead of a generic "Ready" string, and
  `clamp()`-based responsive font sizing.
- Added Scan and Bandwidth fields to "Other options"; Delimiter+RTC, IP-type+Mode, and
  Group+Scan now share rows (RTC/Scan converted to checkboxes, shrunk to their own content so
  tapping empty space no longer toggles them by accident).
- "Other options" merged into the Proxy card as a collapsible panel (chevron at the card's
  bottom edge) with a per-client RAM/CPU/uptime display for its own proxy instance.
- New "IP Public" live check: reports the client's actual exit IP (through its proxy, or its
  WAN egress by fwmark) fired automatically after the page loads.
- Fixed self-service bandwidth/routing changes not taking effect at runtime (DB-only save
  previously never triggered nftables/tc/dhcp) — conditional runtime-apply triggers now mirror
  the desktop's own `runtimeChanged()` checks.
- Fixed NAT-forwarded traffic (e.g. ADB on a port-forwarded client) getting redirected into
  tproxy when that same client is in proxy/mixed mode — added an early
  `ct state established,related return` so already-established connections' reply traffic is
  never mistaken for a fresh outbound flow needing the proxy.
- New `mobile_hide_mode_enabled` setting (Clients tab → ⚙ Setting): when on, hides the Mode
  selector on the mobile self-service page entirely — only an admin logged into the web app can
  change a client's Mode.
- Moved the Clients tab's "Làm tươi"/"Làm tươi proxy" refresh-interval controls into the same
  ⚙ Setting popup as the settings above.

## v0.3.0 — 2026-07-09

Full router-engine port from `proxy_web_debian` (Python) to `daomai-agent` (Go), Phases 1-10:

- Storage foundation: SQLite-backed CRUD for clients, proxies, interfaces, egress, PPPoE
  accounts, and DNS Proxy (DDNS) targets.
- nftables generation (fwmark-based egress routing, no-internet drop set) with an explicit
  generate-then-confirm apply step — nothing is applied to the kernel automatically.
- PPPoE dial-up config generation + rotate (admin-authenticated and public unauthenticated,
  rate-limited) endpoints.
- Proxy hosting via sing-box (tproxy for upstream proxies, local listeners for "PPPoE as proxy"),
  plus a batched systemd-based health check.
- Per-client bandwidth shaping (tc HTB + IFB-based upload shaping).
- DNS Proxy (DDNS) auto-update: a 5-minute background loop fires each target's `update_url`
  bound to its egress's fwmark.
- Scan (LAN isolation): blocks a flagged client from reaching other LAN-phone peers while
  keeping the router's own gateway IP reachable.
- Client online/offline detection: ARP/neighbor-table reachability with a concurrent ping
  fallback for LAN-to-LAN traffic the router's own ARP cache never observes.
- Mobile self-service page: a peer-IP-resolved, no-login page for a device to view/edit its own
  proxy assignment.
- A new admin web UI (Clients, Proxy, Internet, DNS Proxy, Áp dụng/Generate, Hệ thống tabs),
  restyled to match the production router's visual identity (colors, Chrome-style tab strip,
  panel cards).
- Fresh-install interface auto-sync: every physical NIC is seeded with role `lan_phone` on first
  boot (via Go's stdlib, no shelling out to `ip`) so a stock image is reachable out of the box,
  without overwriting an already-classified interface.

## v0.2.0 — 2026-07-09

- Web UI moved to port 18080 (18081 reserved for a future PPPoE-rotate API).
- `POST /api/admin/password` — change the root password from the web UI.
- `POST /api/ssh/enable` — open SSH on an admin-chosen custom port (never 22, never < 1024) by
  rewriting `sshd_config.d` and enabling/restarting `ssh`.
- `GET /api/ssh/status` — report the currently configured SSH port and whether it's active.
- `daomai-agent` can now self-update: `GET /api/update/check` / `POST /api/update/apply` download
  the latest release `.zip` from this repo, back up the current binary + web assets, apply the new
  ones, roll back on failure, and restart itself. `/etc/daomai/config.yaml` is never overwritten by
  an update.

## v0.1.0 — 2026-07-08

- Initial placeholder `daomai-agent`: `/api/status`, `/api/interfaces`, `/api/routes`, `/api/nft`.
- Boot-time forced root password change (`chage -d 0 root`) if never set.
