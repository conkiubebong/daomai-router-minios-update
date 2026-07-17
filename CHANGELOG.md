# Changelog

## v0.4.23 - 2026-07-17

PPPoE boot recovery release.

- Fixed a toram/live-boot reboot gap where generated PPPoE peer files and `daomai-*.service` units disappeared after power loss, but boot restore did not regenerate/start them from the persisted router database.
- Clears stale PPPoE `runtime_ifname`/online state during boot restore before reconnecting, so policy routing/proxy/nft generation does not render against an old `ppp0` from the previous boot.
- Boot restore now starts enabled PPPoE services before proxy/nft/tc reconciliation; the PPP `ip-up` hook then records the live `pppX`, public IP, route table default, and online status.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `044122db`.

## v0.4.21 - 2026-07-17

Update zip packaging fix.

- Repacked the v0.4.20 NAT/DDNS update with Linux-safe zip entry paths (`web/...` instead of Windows-style `web\...`) so the router updater can find the top-level `web/` directory after extraction.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `6b6cf69e`.

## v0.4.20 - 2026-07-17

Realtime NAT/DDNS update test release.

- Added an enable/disable switch to each per-port NAT row in the client NAT popup. Disabled rows stay saved but are hidden from active `OUTPUT_NAT`.
- Public PPPoE rotate now polls the runtime PPP interface after restart and updates the stored Public IP when the new address appears.
- DNS NAT/DDNS now checks the mapped NAT IP every 5 seconds and triggers the configured update URL when the IP changes.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `f78e5c05`.

## v0.4.19 - 2026-07-17

Self-update channel and PPPoE runtime repair release.

- Fixed `daomai-agent` checking the old `conkiubebong/daomaios-update` GitHub Releases repo. It now checks `conkiubebong/daomai-router-minios-update`, which is the actual update zip release repo.
- Fixed PPPoE reconnects after DNS/NAT changes rendering NAT, proxy, load-balance, and policy-route state against the physical NIC (`enp7s0`) while the PPP runtime interface (`ppp0`) was temporarily down.
- PPPoE ip-up/ip-down hooks now reapply proxy/NAT/routing state when the live PPP interface changes, keeping router DNS/DDNS/update traffic on the restored PPP default route.
- Bundled the current DaoMai router agent and web UI from `daomai-router-minios` master.

## v0.4.11 — 2026-07-12

First cluster of the minios->daomaios rebrand, deliberately scoped to low-risk, text-only
changes (the bootloader/LIVEKITNAME/ISO-identity rename that broke boot in an earlier attempt
is a separate, higher-risk cluster, not attempted here).

- Fixed the login page still showing "Mac dinh: admin / admin" and pre-filling the username
  field with "admin" -- stale leftover from before login switched to authenticating against
  the real Linux root account. Now correctly shows/pre-fills root/daomai.
- Renamed purely cosmetic "MiniOS" mentions in Go code comments and the self-update HTTP
  client's User-Agent string to "DaoMaiOS".

## v0.4.10 — 2026-07-12

Multi-WAN internet fix, port uniqueness, faster client discovery, bandwidth shaping fix. Built on
top of v0.4.8 (pre-rebrand base).

- Fixed clients getting no internet on a freshly-assigned dhcp_wan/static_wan egress
  (DNS_PROBE_FINISHED_NO_INTERNET): the real `/apply/nftables` handler never set up
  per-egress policy routing at all (an earlier attempt had fixed a different, unreachable
  dead-code copy of the handler by mistake); nftables also incorrectly gated a client's
  internet access on its egress's momentary health-check status instead of just "does it
  have a real, non-disabled egress" (matching the Python reference).
- Fixed the gateway-fallback gap for fresh dhcp_wan/static_wan egresses whose own gateway
  column hadn't been populated yet.
- Fixed the "ID PORT NAT" DNS-Proxy assignment silently never saving (a decoding bug meant
  it was dropped on every save); duplicating a PPPoE session ("Nhan ban") now correctly
  carries it over to the clone.
- Added uniqueness enforcement for HTTP/SOCKS5 proxy ports (both plain and NAT) so two
  proxies can no longer silently claim the same port.
- PPPoE session delete now cleans up its systemd unit in the background (faster response)
  and no longer leaves it listed as "failed" after removal; deleting a pppoe-type egress
  through the generic delete endpoint now gets the same cleanup as the dedicated path.
- Public PPPoE rotate link now logs its systemctl call through the audit trail like every
  other system mutation.
- Client auto-discovery: actively pings the DHCP pool to prime the ARP cache instead of
  only reading whatever's already there, dropped the discovery interval to 5s, and removed
  the manual "Scan LAN"/"+ Client"/"Import CSV" buttons (now fully automatic). Also fixed
  client-list auto-refresh silently freezing whenever any input anywhere on the page was
  focused, not just inside the Clients tab.
- Bandwidth shaping: added fq_codel as each client's HTB leaf qdisc, fixing a real ~20%
  throughput shortfall from TCP's congestion-control sawtooth against the hard rate
  ceiling (burst/cburst sizing alone, from v0.4.9, wasn't enough on its own).
- Reduced the syslinux boot menu's hidden-autoboot wait from ~10s to ~0.1s on BIOS/legacy
  boot (UEFI/GRUB already booted immediately).
- Interface card UI: raw ifname now always shown on its own line below the editable
  display name.

## v0.4.8 — 2026-07-11

Bundled router-agent fixes built on top of v0.4.7 (pre-rebrand base), deliberately skipping the
untested MiniOS->DAOMAIOS boot-splash rebrand, which caused a real "failed to load ldlinux.c32"
boot failure on real hardware that's still under separate investigation. Supersedes v0.4.9 as the
recommended build until that rebrand issue is resolved.

- Fixed the LAN-phone bridge (`br-phone`) silently losing its member interfaces and going
  carrier-DOWN a few minutes after a successful apply -- NetworkManager was re-claiming
  `lan_phone`-role interfaces out from under the bridge. `applyLANPhoneBridgeCore` now tells
  NetworkManager to release a port (`nmcli device set <if> managed no`) before enslaving it
  into `br-phone`, and hands it back (`managed yes`) when a port leaves the `lan_phone` role.
- Bandwidth shaping now bypasses LAN-to-LAN traffic on `br-phone`/`ifb-daomai`, only capping
  traffic that actually goes to/from the internet through the router's own LAN-phone gateway IP.
- Hid agent-managed virtual interfaces (`br-phone`, `ifb-daomai`) from the Internet tab's
  interfaces list; they no longer show a spurious role combobox/settings button.

## v0.4.9 — 2026-07-11

- Fixed the LAN-phone bridge (`br-phone`) silently losing its member interfaces and going
  carrier-DOWN a few minutes after a successful apply -- NetworkManager was re-claiming
  `lan_phone`-role interfaces out from under the bridge. `applyLANPhoneBridgeCore` now tells
  NetworkManager to release a port (`nmcli device set <if> managed no`) before enslaving it
  into `br-phone`, and hands it back (`managed yes`) when a port leaves the `lan_phone` role.

## v0.4.7 — 2026-07-11

- Cosmetic-only rebrand: the boot-time console banner text now reads "DAOMAIOS" instead of
  "MiniOS". Note the ISO output filename still uses the internal "minios" identifier
  (deliberately not renamed -- deeply coupled to the boot process across two initramfs
  implementations, GRUB/EFI paths, and the ISO volume label; a full rebrand needs its own
  careful incremental pass with real boot-testing after each step).

## v0.4.6 — 2026-07-11

- Default root password is now `daomai` (works for both console/SSH login and the Web UI, since
  they share the same Linux root password). SSH is now LAN-only by default in nftables -- WAN
  access requires an explicit opt-in NAT via the Internet tab's new "Remote SSH access" panel,
  mirroring the existing Remote Web UI/Public API NAT pattern.

## v0.4.5 — 2026-07-11

- Mobile self-service now runs on its own dedicated port (18082) instead of being distinguished
  by IP-matching on the same port as the admin UI (18080) -- fixes a bug where a fresh
  single-NIC box's only reachable IP (the LAN-phone gateway) made the admin UI unreachable,
  always showing mobile.html instead. Accessing the router's gateway IP on port 80 always shows
  mobile self-service; port 18080 always shows the admin UI.

## v0.4.3 — 2026-07-10

- Boot menu now skips the interactive language-selection screen (`minios-cmd -mln en_US` instead
  of the default `multilang`) -- boots straight to the English boot menu.

## v0.4.2 — 2026-07-10

- Console tty1 now auto-logs into root (`agetty --autologin`) instead of forcing a first-login
  password change via `daomai-force-password-reset` — that service is still shipped but no longer
  enabled by default, so a deployment wanting the forced-reset security posture back can
  `systemctl enable --now daomai-force-password-reset` manually.

## v0.4.1 — 2026-07-10

- Added `restoreRuntimeStateOnBoot` (`cmd/daomai-agent/main.go` / `router_api.go`): every
  daomai-agent start (reboot OR crash-restart, since the systemd unit has `Restart=always`) now
  re-applies LAN-phone bridge/DHCP, proxy reload, nftables, tc shaping, and NIC tuning from its
  own database — fixes clients not being recognized after a reboot, since the only prior
  boot-time restore mechanism (`daomai-apply.service`) ran Python's own `boot_restore.py`
  against a completely separate, stale database.
- Changed the pre-boot-restore nftables baseline (`/etc/nftables.d/daomai.nft`) forward chain
  from `policy accept` to `policy drop`, so nothing can leak/forward through the router between
  early boot and daomai-agent finishing its own apply — fail closed instead of fail open.

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
