# Changelog

## v0.4.48 - 2026-08-01

Fix mobile bandwidth display bug + add self-service local IP change.

- Fixed mobile self-service silently showing a bandwidth limit of "5" for a client whose real, stored limit is "0" (unlimited) -- the admin web app already showed the correct value; mobile's response builder was rewriting 0 into a hardcoded 5 before sending it.
- Added a "Local IP (LAN)" field to the mobile self-service page: a client can now request its own static IP reservation directly instead of only toggling static/dynamic. The new address is validated (a real IPv4, inside the router's LAN subnet, not the router's own gateway address) and duplicates against another device are rejected with a clear error.
- Fixed a real gap that would have made the new IP field silently not work on reconnect: self-service's runtime-change detection only re-applied dnsmasq (regenerate + prune the stale lease + restart) when `ip_type` or the name changed, never when only the IP value changed. A phone that changed its IP would keep getting its OLD address until this was fixed. The admin web app's own client-edit path already had this right; self-service now matches it.
- After a successful local-IP change, the page now tells the user to toggle WiFi off/on to pick up the new address, instead of confusingly showing "device not registered" (the phone's own connection is still using its old address until it reconnects).
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0c2603f2`.

## v0.4.47 - 2026-08-01

Split "flash to a different disk" into its own panel + fix stale cached JS.

- The "Cap nhat ISO" panel goes back to its original behavior: no disk picker, it always auto-detects and re-flashes whatever disk this session actually booted from.
- Added a separate "Ghi ISO vao o khac" panel with its own disk picker, offering two independent ways to write a chosen disk (e.g. an internal HDD/NVMe): download+verify the latest release ISO and write it there, or clone the currently-booted USB straight across via `dd` with no internet download or SHA256 check at all -- useful when there's no network access, or when you just want an exact copy of the router you're already running (including its current config/router.db).
- Fixed a real bug affecting every browser session opened before the v0.4.44/v0.4.46 releases: the admin page's JS cache-busting version string for `08-system-misc.js` was never bumped across those two releases, so those sessions kept running pre-redesign JS against the new backend -- showing "HTTP 404" on Backup and missing the ISO disk picker entirely. All JS cache-bust versions are now refreshed; affected sessions just need one hard refresh (or will pick it up automatically once this version is installed).
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `9b0623fa`.

## v0.4.46 - 2026-08-01

Flash ISO to a different disk.

- The Update tab's ISO section can now write to a disk other than the currently-booted USB. A new disk picker combobox lists every disk the router sees (size, model, and whether it's the current boot USB or already has data/mounted), defaulting to the current boot disk so nothing changes unless you pick something else. This lets you write the router image onto an internal HDD/SSD instead, so the router can boot from that disk without the USB stick plugged in.
- Writing to a disk other than the current boot disk safely skips the `router.db` backup/restore steps that only make sense for re-flashing in place -- there's nothing existing on a fresh disk to protect, and the currently-running system's own persist partition is never touched.
- The target disk is validated against the live disk list before anything is written, and picking a disk other than the boot USB shows a separate, explicit warning naming the exact disk that will be erased.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `e57d0217`.

## v0.4.45 - 2026-07-31

Multiple DNS Proxies (DDNS hostnames) per egress.

- The DNS Proxy tab no longer limits an egress to one DDNS hostname: a DNS Proxy is really just an independent DDNS registration for that egress's current public IP, and a real need surfaced -- tracking one egress with two different DDNS hostnames at once (e.g. ahead of adding a second PPPoE line, so one hostname can move to it later without touching the first). Removed the constraint that used to reject this with "dns proxy egress_id already exists"; it was never load-bearing for the actual auto-update mechanism, which already updates every DNS Proxy row independently.
- Includes an automatic one-time migration that rebuilds the `dns_proxies` table on the next boot for already-deployed routers, preserving all existing data -- no manual steps needed after updating.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0701a2ad`.

## v0.4.44 - 2026-07-30

Backup/restore redesign + lower persist partition floor.

- Lowered the persist partition's minimum trailing free space from 512MB to 64MB: the old floor was an arbitrary sanity check, not based on actual database size (router.db is a few MB even under heavy use). 500MB-1GB RAM installs can now get a working persist partition instead of being rejected outright.
- Backups are now strictly manual: removed the automatic "pre_X" safety backups that used to be taken before every PPPoE/LAN-phone apply operation.
- Simplified the backup format to a single raw `.db` file: "Backup" now streams a fresh snapshot straight back as a download the moment you click it -- nothing is kept on the router, so nothing can silently accumulate and fill the disk (the old zip format also bundled `generated/` config files, which are pure derived artifacts already regenerated from the database on every boot).
- Rewrote and fixed Restore: pick a `.db` file from your computer -- it's validated as a genuine SQLite database, written to both the live running database and the persist-partition copy at the same time, then the router reboots immediately to apply it. This closes a real race where an untimely reboot/power loss shortly after a restore could silently undo it. Also fixes the previous Restore button always failing outright with an error (the frontend never sent a parameter the backend required).
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `2c1eeddf`.

## v0.4.43 - 2026-07-30

Proxy delete cleanup + immediate proxy-auto pool pick.

- Fixed a proxy delete leaving referencing clients broken: `clients.proxy_id` had no `ON DELETE` clause, so deleting a proxy still assigned to a client (manual or "proxy auto") left that client with a permanently dangling `proxy_id` -- its Proxy cell in the admin UI just went blank with no error, silently losing its proxied traffic path. Deleting a proxy now clears `proxy_id` on every client still referencing it first.
- Fixed "proxy auto" silently doing nothing when the client already had a live (non-dead) manually-assigned proxy: switching it to "proxy auto" via the mode dropdown used to keep it stuck on that same old proxy indefinitely instead of ever drawing from the pool, since the sticky "keep unless dead" rule doesn't distinguish a pool pick from a leftover manual one. Enabling "proxy auto" now forces one immediate pool reassignment, in both the admin app and mobile self-service.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `071fbe11`.

## v0.4.42 - 2026-07-26

Hot-update stability and ISO follow-up fix.

- Fixed a race after installing an update zip: the old agent process could run one more integrity-watchdog pass before `systemctl restart daomai-agent` stopped it, see the newly installed binary/Web UI as mismatched against its old embedded baseline, and reboot the toram system back to the old ISO.
- After a zip update succeeds, the ISO download/write controls are immediately re-enabled from the current release metadata so users can still flash the matching ISO even when the running version already equals the latest release.
- Repacked the update zip with POSIX `web/...` paths so Linux extraction creates the expected `web` directory.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `817d5459`.

## v0.4.41 - 2026-07-26

ISO update reboot choice.

- Changed the ISO update flow so writing the ISO and rebooting are separate decisions: after the verified ISO is written to the boot disk, the UI asks whether to reboot now or keep running until a later manual reboot.
- `/api/update/iso/apply` now accepts `reboot=false`; old callers that omit it still reboot automatically for compatibility.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `ab068bbe`.

## v0.4.40 - 2026-07-26

DHCP stale-client cleanup runtime fix.

- Fixed dynamic DHCP clients not being cleaned up on fresh/default installs: the backend did not seed `static_ip_timeout_minutes` and `static_ip_reaper_enabled`, even though the UI showed the cleanup as enabled with a 30-minute timeout.
- Fixed stale runtime policy after an expired dynamic client is deleted: the reaper now regenerates and applies nftables and tc shaping immediately, so the deleted client's internet role and bandwidth limits do not remain attached to the old IP when DHCP later reuses it.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `64400b2b`.

## v0.4.39 - 2026-07-24

Three broken button fixes.

- Fixed the "Đổi mật khẩu" (Change Password) topbar button: it had no click handler at all since the web UI was rewritten as a split-file SPA, so clicking it did nothing and there was no way to change the admin password from the web UI. Now wired to `POST /api/admin/password`, with client-side new/confirm password mismatch validation and success/error feedback shown in the modal.
- Fixed the System/DHCP "Apply" button: clicking Cancel on the confirmation dialog still sent the apply request to the backend (the backend correctly rejected it without applying anything, but the button now stops immediately on Cancel like every other destructive-action button in the UI).
- Fixed the "Check IP" button: it silently did nothing when the proxy ID field was left empty. Now shows an inline error.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `2e627600`.

## v0.4.38 - 2026-07-24

Dynamic-client reaper fix.

- Fixed the "Auto-reap expired dynamic IPs" DHCP setting: it never actually deleted offline dynamic clients. It compared each client's original creation time against the configured timeout instead of how long the client had actually been offline, and its `LastSeen`-empty check was practically unreachable since every dynamic client gets `LastSeen` stamped immediately on creation. A dynamic client that connected once and later dropped offline for good would linger in the clients table forever, inflating the Dashboard's total client count and the DHCP client list. Now the reaper uses each client's real last-online timestamp (falling back to creation time only if it was never confirmed online) and only reaps clients that are actually offline.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0dcb22f0`.

## v0.4.37 - 2026-07-24

Dynamic IP reclaim, static IP apply, and ip_type guard fixes.

- Fixed a serious bug: when a dynamic DHCP lease was reclaimed from a stale/offline client and handed to a new device, the new device inherited the previous device's internet access, mode, and bandwidth settings instead of getting the default new-client policy (e.g. `No_Internet`). The DHCP lease importer now deletes the stale dynamic claimant and lets the new MAC get a clean client record under whatever default policy is configured, rather than silently rewriting the existing record's identity.
- Fixed a serious bug: changing a static client's IP address in the admin UI had no real effect on the network -- the physical device kept getting its old IP from dnsmasq even after a WiFi reconnect, because the live dnsmasq config was regenerated to disk but never actually installed/reloaded. Client create/update now synchronously applies the full LAN-phone DHCP config (regenerate, validate, install as the live config, restart dnsmasq, prune stale leases) whenever a static client's IP/MAC/name changes.
- Added a clear rejection instead of a silent revert: setting a client's `ip_type` back to `dynamic` while a proxy or NAT port forward is still attached is now rejected with an explicit error, in both the admin API and mobile self-service, instead of being silently ignored or inconsistently allowed.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `cfddbcef`.

## v0.4.36 - 2026-07-22

Mobile self-service proxy_line validation fix.

- Fixed a live bug: saving the Mobile Self-Service page with the Internet dropdown set to `No_Internet` (or any mode that doesn't need a proxy, e.g. `blocked`/`direct`) wrongly failed with "proxy_line required". The mobile form always resends the current proxy-line input's value on every save regardless of what the user actually touched, and the backend rejected any empty value unconditionally. Now only requires a non-empty proxy line for modes that actually need an upstream (`proxy`/`mixed`).
- A client that switches off `proxy_auto` this way (empty proxy line, mode changed to blocked/direct) is now also correctly unenrolled from the proxy pool, instead of staying flagged and keeping getting reassigned pool proxies it no longer wants.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `d93b59aa`.

## v0.4.35 - 2026-07-21

File-integrity watchdog (self-heal via reboot).

- `daomai-agent` now MD5-checks its own Web UI static assets (`index.html`, `static/**`: js/png/css/swagger) and its own binary every 30s, against a pristine baseline baked directly into the binary at build time (Web UI via `go:embed`, the binary's own hash sealed by a build-time patch step) -- not a separate file an attacker could edit to match a tampered build. A first-pass violation is rechecked after 3s before being treated as confirmed, to rule out reading a file mid-write.
- On confirmed tamper, reboots the router: because DaoMaiOS boots `toram` (the whole OS runs from RAM), a reboot genuinely restores every file from the pristine boot image. Reboots are rate-limited by a cooldown persisted across reboots, so a persistent attacker or false positive can't fast-loop the hardware.
- The watchdog automatically pauses during a legitimate hot update (`Update zip`) so it never false-positives on its own in-place binary/web swap; a release's own agent binary always carries the baseline matching its own bundled web files.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0fa5086b`.

## v0.4.34 - 2026-07-18

Mobile self-service port-80 redirect scoping fix.

- Fixed a real live bug: browsing to any plain `http://` site (e.g. `http://deviceinfo.me`) on a LAN-phone client redirected to the router's own mobile self-service page instead of the actual site, while `https://` sites worked normally. Root cause: the nftables rule redirecting port 80 to the self-service portal was unscoped -- it hijacked every port-80 request regardless of destination, not just requests aimed at the router's own gateway IP. Scoped the rule to `ip daddr <router IP>` so arbitrary HTTP browsing now passes through untouched; only requests actually aimed at the router itself (or the existing blocked-client self-service escape hatch) still redirect.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `ff53ab0d`.

## v0.4.33 - 2026-07-17

PPPoE PMTU blackhole fix.

- Fixed a real connectivity bug reported live: Google/Facebook loaded fast, but other sites (e.g. GitHub) either hung or rendered with missing CSS/JS. Classic PPPoE PMTU blackhole -- the pppoe link's own MTU was already correctly set to 1492, but that alone never clamped a LAN client's own TCP handshake (negotiated off the client's own 1500-MTU interface) for forwarded traffic, so any response requiring a full-size segment silently vanished wherever ICMP "fragmentation needed" gets filtered upstream (very common). Small responses (basic HTML, QUIC/HTTP3) never hit this, which is why some sites appeared to work fine.
- Adds TCP MSS clamping (`tcp flags syn tcp option maxseg size set rt mtu`) to the forward chain -- self-adjusts per route/egress type, not a hardcoded value. Verified live: applied cleanly and confirmed present in the running kernel ruleset.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `2baed637`.

## v0.4.32 - 2026-07-17

DNS Proxy nftables reapply fix.

- Fixed the DNS Proxy panel's save/delete actions never re-generating/re-applying nftables: after v0.4.30's fix made a DNS Proxy's own egress selector actually control a linked proxy's WAN NAT rule, changing that selector either way (detaching to No_Nat, or re-attaching to a real egress) had no live effect until the agent happened to restart -- the kernel kept running whatever ruleset was already loaded. Verified live: re-attaching "nat_01" to a real pppoe egress now correctly restores external access again.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `241b7263`.

## v0.4.31 - 2026-07-17

Clients tab "Group" filter release.

- Added a "Group" hide/show filter next to the existing "Columns" one in the Clients tab: same instant/persisted mechanism (one CSS rule + localStorage, no "Apply" click) but per row instead of per column, so a long client list can be decluttered down to just the groups you care about. Independent of the existing single-group filter in the Filter panel.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `e769b4aa`.

## v0.4.30 - 2026-07-17

DNS Proxy NAT fix + mobile proxy-auto + system clock release.

- Fixed a real security-relevant bug: setting a DNS Proxy ("id nat" target, e.g. "nat_01") back to No_Nat in the DNS Proxy panel still let every proxy pointed at it be reached from the WAN -- the panel's own egress selector never actually fed into the "Proxy NAT port" nftables rule at all. Now clears a proxy's WAN NAT ports when its DNS Proxy target is detached, both in the preview and live-apply paths.
- Mobile self-service now supports "proxy auto" devices: a new "proxy auto" mode option locks the proxy line-edit/type read-only and shows a rotate button next to it, calling the exact same pool-rotation logic (dead-proxy skip, per-proxy device cap) as the admin web UI's own rotate button -- not a reimplementation.
- Added real NTP time sync (systemd-timesyncd) and clock persistence across reboots (fake-hwclock, saved every 15 min). The router previously had neither installed at all: a dead/absent RTC (seen live reading back 2020) combined with no time source left the system clock sitting arbitrarily wrong after a toram boot, with nothing to correct it -- causing an outright TLS "certificate not yet valid" failure on the update-check HTTPS request (and any other HTTPS call) even though the network path itself was fine.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `37b01fda`.

## v0.4.27 - 2026-07-17

ISO download/flash split release.

- Added `POST /api/update/iso/prepare`: downloads and SHA256-verifies the latest release's ISO into a pending location without ever touching the boot disk, and `GET /api/update/iso/prepare/status` for the Web UI to poll a live download progress bar.
- `POST /api/update/iso/apply` now only writes an already-prepared, already-verified ISO -- it refuses to run and errors clearly if no ISO was prepared first, instead of downloading and writing in one uninterruptible call.
- The Web UI's ISO button now downloads first, shows real progress, then pops a second confirmation once the download is verified before actually flashing the disk. Declining that confirmation keeps the verified ISO ready to flash later without re-downloading.
- Allows re-flashing the ISO even when `current == latest` (a prior zip hot-update can already have bumped VERSION without ever touching the booted image).
- Fixed the `br-phone` stale gateway IP after a `dhcp_gateway` subnet change, and a misleading ISO-update status message.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `33aa5461`.

## v0.4.25 - 2026-07-17

br-phone gateway bug fix release.

- Fixed `ip addr replace` only replacing an address when it exactly matches the previous one (same IP + prefix). Moving `dhcp_gateway` to a different subnet left the OLD address live on `br-phone` alongside the new one, so the router kept answering DHCP on both subnets -- phones just kept renewing against the still-reachable old gateway and never picked up the change, even after toggling Wi-Fi. Now flushes the bridge's IPv4 addresses before applying the new one.
- Fixed the ISO-update panel showing "no .iso available" instead of "already up to date" once `current == latest`.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `622242a6`.

## v0.4.24 - 2026-07-17

ISO self-flash update release.

- Added a full ISO self-update path: `POST /api/update/iso/apply` on `daomai-agent` verifies the release ISO's SHA256, backs up `router.db`, `dd`s the ISO over the boot disk, restores the persistence partition, then reboots -- an alternative to the zip hot-update for changes that need a full OS re-flash.
- This is the first release to publish `daomaios_<version>.iso` + `.iso.sha256` as GitHub Release assets alongside the existing update zip, specifically so `/api/update/iso/apply` has something to fetch and verify.
- Removed the redundant PPPoE username line from each session's summary card on the Internet tab; it's already editable in that session's config form.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `4b5d8cfb`.

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
