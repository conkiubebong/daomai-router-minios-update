# Changelog

## v0.4.74 - 2026-09-07

Keep configuration on media written from the ISO, add a one-URL proxy API for tools, and cut the background write/restart load.

- Media written from the ISO (Rufus "ISO mode" and friends) saved nothing at all, while media written with `dd` saved fine. Such a writer creates its own FAT32 partition spanning the whole stick, so the trailing free space the dedicated persistence partition is carved from does not exist. The agent now falls back to a folder on the boot medium's own filesystem, and failing that to a dedicated partition on an INTERNAL disk -- only ever claiming free space at the end of a disk, never touching an existing partition. A disk with no partition table at all is given one only after confirming it carries nothing.
- A self-triggered ISO update on such media could leave the stick unbootable. The update unmounted one hardcoded path, but in the boot-filesystem fallback the live mount is on the very partition `dd` then rewrote -- still mounted and dirty throughout, so the kernel flushed cached blocks back over the freshly written image afterwards. It now unmounts whatever the active mode is really using.
- "Mounted" and "writable" are not the same thing: a FAT32 or NTFS volume Windows left dirty mounts without complaint and then rejects every write, which the agent reported as healthy persistence. It now writes a probe file before trusting a location, and falls through to the next option when that fails. `/api/status` gained a `persistence` block (mode, writable, device) so this stops being a startup log line nobody reads.
- A restored database is validated before it overwrites the live one. A damaged copy used to be restored blindly, after which the agent failed to start, and `Restart=always` turned one bad file into a router that never finished booting. It is now moved aside and the agent starts from an empty database instead.
- The DHCP pool is validated. `dhcp_range_start`/`dhcp_range_end` were stored with no checks at all while the gateway beside them was validated, so a typo took out LAN DHCP *and* DNS (dnsmasq serves both), and a pool outside the bridge's own subnet passed `dnsmasq --test`, started cleanly, then answered every client with "no address range available".
- PPPoE credentials are escaped for pppd's config format. A password containing a backslash or a double quote -- both routinely issued by ISPs -- was written through verbatim and silently truncated, so the session failed to authenticate with nothing anywhere pointing at the password.
- New tool-facing API to point one phone at one upstream proxy with a single URL, addressing the phone by IP, MAC or name: `/api/public/v1/phones/{selector}/proxy/socks5/HOST:PORT:USER:PASS` (query form and a compact `/api/public/proxy/...` alias also accepted). Calling it again for the same phone edits that phone's existing proxy instead of piling up a new record per call. The endpoints an integrator actually opens the API page for -- this one and the PPPoE rotate/restart/toggle calls -- now sort first there.
- The Info tab now shows where configuration is actually being saved, in red when nothing durable is available or the location is mounted but not writable. Losing persistence is silent by design: everything works normally right up until the reboot that throws the work away.
- Background load cut in three places. The client status refresher rewrote `last_seen` for every online client every 5 seconds, one transaction each -- twenty database commits a second on a hundred-client LAN, onto the USB stick the router runs from, which also kept the database permanently dirty so the whole file was re-snapshotted every 30 seconds around the clock. Proxy reloads restarted every sing-box instance at once, a spike large enough to make already-healthy proxies stutter on a box running many. The Internet tab probed every session in parallel every 5 seconds with no guard against rounds overlapping, and those requests queued ahead of the admin's own clicks in the browser's per-origin connection limit -- the real cause of the UI feeling frozen with many duplicated PPPoE sessions.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `938bc80e1f4a6b38079b7daba4a556aee49bb5ba`.

## v0.4.73 - 2026-08-23

Fix UEFI machines failing to boot after a self-triggered ISO update.

- Writing a new ISO onto the currently-booted disk in place with `dd` changes that disk's partition UUIDs/signature even though the file contents are otherwise identical to before. Some UEFI firmware pins its existing NVRAM boot entry to the OLD UUID, so after a self-triggered "Update ISO" that entry still shows in the firmware's boot menu (the label is stored separately from the UUID it targets) but fails to actually resolve when selected, bouncing back to firmware setup instead of booting -- confirmed live on a real UEFI machine, while writing the identical image externally via Rufus/Etcher (never touches NVRAM) was unaffected. The agent now refreshes the UEFI boot entry right after writing the new ISO: deletes any of its own prior entry, then creates a fresh one pointing at the newly-written EFI System Partition. Best-effort only -- never fails the update, and does nothing on non-UEFI machines.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `87665c44c036591087f64335226c05fd4e4cfb88`.

## v0.4.72 - 2026-08-23

Split Viettel VLAN handling, fix dhcp_wan local IP sync, stop auto-creating placeholder PPPoE sessions.

- The interface ISP dropdown forced "Viettel" to always mean "needs a tagged VLAN sub-interface", hiding the session-management UI entirely until a VLAN was created -- live testing confirmed Viettel can also dial directly on the physical port (no VLAN needed) given a unique per-session MAC. Split into "viettel" (direct) and "viettel_vlan35" (needs VLAN first); both still get macvlan MAC virtualization, only the VLAN requirement differs.
- `egress.local_ip` for `dhcp_wan` never got synced from the live DHCP lease -- confirmed live, switching a port to `dhcp_wan` left the egress form's Local IP field permanently blank despite a real lease being active. `applyPolicyRouting` now syncs it, mirroring the existing gateway fallback.
- Switching an interface to `pppoe_wan` and saving used to retype a leftover egress in place, auto-materializing an empty, credential-less PPPoE session card with no way to make it not appear short of deleting it again. A pppoe egress now only ever comes from the explicit "+ Thêm phiên PPPoE" action.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `e39928b403a5d59c678da5be0b84f1bf57063438`.

## v0.4.71 - 2026-08-22

Fix client traffic leaking through the main routing table for a disconnected egress.

- `applyPolicyRouting` used to skip a pppoe egress entirely (no `ip rule`/route table installed at all) whenever it had never dialed successfully. nftables' `mark_egress` still fwmarks that egress's clients' traffic regardless, so with no matching policy-routing rule, Linux silently falls through to the next rule (the main table) instead of failing outright -- if the main table happens to have any other default route, that client's traffic leaks out through it instead of correctly having no internet, since Linux policy routing has no "no route for this mark" failure mode. A disconnected pppoe egress now still gets its `ip rule` installed, with an explicit `ip route replace unreachable default table <N>` placeholder so fwmark'd traffic is definitively rejected instead of silently leaking elsewhere. Once pppd actually connects, the existing ip-up hook's own route replace naturally overwrites this placeholder with the real one.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `90b2d18fcc2a2c48adfc4b3cb478c8bf579abdb8`.

## v0.4.70 - 2026-08-22

Fix false-online PPPoE status and a misleading DDNS rotate error.

- `egresscheck` used to trust a successful public-IP probe even for a pppoe egress with no live runtime interface. A pppoe session whose pppd never actually connected has no ip-up-hook-installed policy-routing rule of its own yet, so a SO_MARK'd probe for that egress's fwmark silently fell through to the kernel's main routing table (Linux policy routing has no "no route for this mark" failure) and could report success via a completely unrelated network path -- confirmed live: a PPPoE session stuck retrying PPPoE Discovery forever still showed "online" with a public IP. A probe success for a pppoe egress with no live runtime interface is now never trusted, regardless of what the probe itself returned.
- The PPPoE rotate button's best-effort DDNS update-IP call now skips entirely when the associated DNS Proxy has no `update_url` configured (a normal, valid state) instead of surfacing a "no update_url configured" error toast unrelated to whether the rotate itself succeeded.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `3cc7e73c7cc1267b065f96702809ad74faaf609a`.

## v0.4.69 - 2026-08-22

Fix PPPoE macvlan reliability and MAC display, add ISP auto-inheritance, enable NAT hairpin by default.

- `applyPPPoEAccountLocked`/`duplicatePPPoESessionOne` used to silently ignore a failed macvlan device creation and still start pppd and report success -- a duplicated Viettel session that failed to get its virtual interface looked identical to a working one. Now surfaces the real failure instead of pretending it worked.
- The admin UI's "MAC:" display for each PPPoE session card was reading the shared physical/VLAN card's own MAC instead of each account's per-session virtual MAC, so every duplicated session on one card displayed the exact same MAC regardless of the underlying kernel devices -- this looked exactly like (and was mistaken live for) the MAC-virtualization feature itself being broken.
- PPPoE accounts now inherit their ISP tag from the physical/VLAN interface they dial over when not set explicitly on the account itself. Live-confirmed root cause of Viettel MAC virtualization never engaging: `isp` on the interface (the pre-existing control that already requires a tagged VLAN for Viettel) and `isp` on the PPPoE account are two separate fields -- an admin who only ever tagged the interface got every account silently landing with no virtual MAC at all.
- LAN NAT hairpin (loopback) is now enabled by default for new/unset installs instead of requiring an explicit opt-in toggle.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `63d713b245310d9d97343411b0cab7848080a3b6`.

## v0.4.68 - 2026-08-21

Fix ISO-update database backup safety, PPPoE REST locking, and mobile self-service validation.

- Coordinate the periodic persistence loop with ISO updates and use the checked SQLite snapshot path for the pre-flash database backup, preventing a race while the persistence partition is unmounted or rewritten.
- Hold `pppoeMu` in the direct PPPoE REST create, update, and delete handlers so they cannot race apply, duplicate, delete, or finalize operations.
- Relax mobile self-service `proxy_line` validation when a save is not changing the proxy line, avoiding false validation failures for existing proxy or mixed clients.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `c3511b6714d81b84113a9f6a5a60c166ee34d2be`.

## v0.4.67 - 2026-08-21

Fix mobile self-service proxy_line bug, copy PPPoE ServiceName on duplicate, add per-session MAC virtualization for Viettel.

- `mobile_hide_mode_enabled` couldn't persist (missing from the settings allowlist), and even once set, `UpdateSelf` still rejected an empty proxy_line for proxy/mixed clients even though the mode selector was hidden from the user -- this resurfaced "proxy_line required" on every save, including ones that never touched the proxy line.
- PPPoE session duplication now also copies `ServiceName`: some ISPs (confirmed live: Viettel, via a real Mikrotik reference config) key PPPoE session admission on the PADI Service-Name tag, and a clone that silently dropped it back to empty failed to dial on those lines.
- Added per-session MAC virtualization (macvlan), scoped to `isp = "viettel"` PPPoE accounts only per live confirmation that FPT/VNPT dial the shared physical/VLAN interface fine as-is: each duplicated Viettel session gets its own macvlan device with a random locally-administered MAC instead of sharing the parent WAN port's real MAC. New ISP dropdown on the PPPoE admin form.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `5973872b42e587ff63fabca65ba45d90aa3a5ff0`.

## v0.4.66 - 2026-08-18

Fix LAN NAT hairpin reply traffic also being policy-routed out the wrong WAN.

- v0.4.65 fixed the forward leg of hairpin NAT (a LAN client's request to the router's own public IP). This release fixes the matching reply leg: the LAN target's reply packet is itself sourced from a registered client with its own assigned WAN egress, so it was being sent down that egress's policy-routing table too -- which has no route back to the LAN -- instead of back to the original requester.
- Confirmed live end-to-end: a raw TCP connect to a RustDesk-style port-forward via the DDNS hostname from a LAN client now succeeds, where it previously stuck at a half-open TCP handshake indefinitely.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `4b9f9c6b38218f0e2b57ffcb5d0d6d69cc7e7873`.

## v0.4.65 - 2026-08-18

Fix LAN NAT hairpin (loopback) traffic timing out despite correct DNAT/masquerade rules.

- The v0.4.64 hairpin feature's DNAT and masquerade rules were both correct, but `mark_egress` (which runs before DNAT, at a higher network-stack priority) was policy-routing a LAN client's hairpin request out that client's own assigned WAN egress table before DNAT ever rewrote the destination to the LAN target -- that WAN table has no route back to the local LAN subnet. `mark_egress` now skips fwmark assignment for the exact (public IP, protocol, port) combinations the hairpin DNAT rule itself targets.
- Confirmed live via a real RustDesk port-forward test; this release fixed the request leg only -- see v0.4.66 for the matching reply-leg fix found in the same test.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `3b2288e3c5218f0e2b57ffcb5d0d6d69cc7e7873`.

## v0.4.64 - 2026-08-19

Add opt-in LAN NAT loopback (hairpin NAT).

- Internet → DNS Proxy now has a switch before “+ Thêm DNS Proxy”. When enabled, LAN clients can reach the router's current public NAT address/DDNS and receive replies correctly.
- Rules are scoped to DNATed LAN flows and the current WAN public IP; default is off.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `06776f19`.

## v0.4.63 - 2026-08-19

Fix NAT popup metadata being dropped during client updates.

- Preserve per-port NAT notes and UDP toggles across the client update database round-trip.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `dbd11410`.

## v0.4.62 - 2026-08-18

Fix PPPoE proxy duplication and NAT popup persistence.

- PPPoE duplication copies the source proxy's UDP relay setting.
- NAT popup notes and UDP toggles persist per port and reload correctly.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `5e032ebf1d23d18d2b5bebe98d0711a66f6322e8`.

## v0.4.61 - 2026-08-18

Add per-port comment and UDP toggle to NAT ports, and a hot-update safety net.

- The per-client NAT ports popup (Clients tab) now has a free-text note field per row (e.g. "RustDesk") and a UDP toggle -- enabling it forwards both TCP and UDP on that port number, matching what RustDesk and similar remote-access tools need.
- `daomai-agent` now verifies a downloaded hot-update zip's agent binary and web UI actually match each other before committing to the swap and restarting. A mismatched release package (an operator packaging mistake, not a router bug -- this is exactly what happened with the initial v0.4.60 zip, since fixed) now fails cleanly with a clear error and automatic rollback instead of silently rebooting the router about 30 seconds after the update appeared to succeed.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0761aa2860218f0e2b57ffcb5d0d6d69cc7e7873`.

## v0.4.60 - 2026-08-18

Fix silent NAT-port data loss on power-loss reboot and a stale NAT ports popup.

- The persist-to-USB snapshot (survives a `toram` reboot) and the manual "Create backup" feature both ran SQLite's `VACUUM INTO` on a separate connection with no `busy_timeout`, so a snapshot could fail instantly with `database is locked (SQLITE_BUSY)` whenever it raced the router's own background writes -- confirmed live on a real unit. Repeated silent failures meant the last successfully persisted snapshot could be much older than expected, so a NAT port (or other recent change) added hours earlier could be missing after an unclean power-loss reboot even though the rest of that client's settings were untouched. Both snapshot paths now set a 5-second `busy_timeout` before `VACUUM INTO`, and the persist retry loop now waits briefly between attempts instead of retrying back-to-back.
- The per-client NAT ports popup rendered from data captured when the client table row was last built, so reopening it after an out-of-band change (including the exact data-loss scenario above) could show stale enable/disable state, or make adding a port that had actually been lost look like a no-op. It now always fetches the client's current data before rendering.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `a8eb49383bfceb70ea55d61fc6ed15c5f6d24d2f`.

## v0.4.59 - 2026-08-16

Add per-client Bypass Proxy: exempt chosen domains from a device's upstream proxy.

- New **Bypass Proxy** tab: paste a device's LAN IP and start a live capture session. The router watches that device's already-proxied connections (via sing-box's own per-connection domain sniffing) and streams visited domains into a log, newest first, with an adjustable row cap and a clear-log control. Any domain can be added to a staging list and saved as a named, reusable bypass list; saved lists can be edited (one domain per line) or deleted.
- New **Bypass Proxy** column on the Clients tab: a checkbox plus a target selector (off / a private custom domain list for that device / any saved bypass list) per device.
- The Internet tab's New Client Policy panel gained a matching Bypass Proxy default, so newly discovered devices can automatically inherit a bypass list.
- Enforcement happens inside sing-box itself: for a bypassed client, matching domains route directly instead of through the upstream proxy, reusing the SNI sniffing already active on the tproxy inbound -- no separate DNS-log tailing or ipset/nftables plumbing required. Capture requires the target device to already be in proxy mode, since only proxied traffic reaches sing-box's sniffing point.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `82e391746c218f0e2b57ffcb5d0d6d69cc7e7873`.

## v0.4.58 - 2026-08-14

Redesign the Internet tab's New Client Policy panel and make new devices usable immediately by default.

- New clients now default to `mixed` mode with dynamic IPs, so both direct and proxied traffic work without an admin editing each device first. Both defaults are configurable in the redesigned panel.
- Removed the redundant "Allow new IPs online" toggle. The exit-route selector now controls the same policy directly: choose `No_Internet` to block new devices, or explicitly choose `blocked` mode when a real default exit route is configured.
- Renamed the exit-route field for clarity and changed the panel to a clean single-column layout for the added mode and IP-type controls.
- The same defaults now apply consistently whether a client is created through the admin API or discovered automatically from DHCP leases and network neighbors.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `36143e61a295a77391d3a658b05f690639e35435`.

## v0.4.57 - 2026-08-14

Add clear confirmation and duplicate warnings when reusing or bulk-importing proxies.

- Mobile Self-Service now detects when an entered proxy is already assigned to other devices and shows their names and IP addresses before saving. Confirming shares the same existing proxy without creating a duplicate; declining leaves the entered proxy line untouched and saves nothing.
- Proxy Pool bulk paste now warns before importing when a proxy is repeated in the pasted list or already exists in the pool, listing each affected `host:port` and its occurrence count. Import behavior is unchanged: duplicates are still skipped.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `bda326b50d92600071249b7180f29ca5d76d0f97`.

## v0.4.56 - 2026-08-11

Fix known DHCP clients losing their saved routing and device settings when they swap IP addresses.

- When two already-known dynamic clients renewed into each other's previous IP addresses in the same dnsmasq lease snapshot, one client could be deleted and recreated with new-device defaults. This silently reset its mode (for example, `proxy` to `direct`), proxy assignment, bandwidth limits, group, comment, and other saved settings. The lease importer now recognizes both clients as relocating within the same snapshot and updates them in place, preserving their existing configuration regardless of lease-file line order; genuinely stale address claimants are still removed normally.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `b9124c29d248b3780f6e1004649a65eefdfe3cbb`.

## v0.4.55 - 2026-08-11

Fix DHCP WAN links getting a valid lease but remaining unable to reach the internet.

- A `dhcp_wan` interface could receive a working IP address and gateway from DHCP yet remain stuck with a gateway-less policy-routing default route, causing all traffic through that WAN to fail. The router now recovers the gateway directly from the interface's dhclient lease when it is missing from both the database and current route, installs the correct `via` route, and saves the learned gateway so it appears in the admin UI and remains available for future reconciles.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `0fa28887ddff93adbf22c731de207fcce070a79c`.

## v0.4.54 - 2026-08-10

Add admin support contact info and improve Dashboard system stats visibility.

- The Web Admin topbar now shows support contact info with a Telegram link and Facebook name, so admins can find the support channel directly from the router UI.
- Dashboard system stats now refresh every 800ms instead of 1500ms for snappier CPU/RAM/disk feedback; the data source is still lightweight `/proc` reads.
- The CPU gauge now also shows the detected CPU model, clock speed, and logical core count from `/proc/cpuinfo`.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `8819feaf150866d313b83855b7217435daed139a`.

## v0.4.53 - 2026-08-10

Fix LAN gateway save breaking DHCP/bridge applies, dhcp_wan watchdog missing stale leases, and other admin-reported bugs.

- `PUT /settings` persisted `dhcp_gateway` with no validation. An unparseable value silently broke every later DHCP/LAN-bridge apply for the rest of the admin's session -- confirmed live. The gateway is now validated before it's saved, and an invalid value is rejected outright instead of corrupting later applies. The DHCP panel in the web UI also now refreshes after a successful save; previously a successful save could look like it had no effect until the admin navigated away and back.
- The `dhcp_wan` background watchdog only forced a renew when an interface had completely lost its IPv4 address, so a stale/expired-but-still-present lease was never noticed or renewed. It now also checks the interface's own dhclient lease-file expiry. Also fixed a race where changing an interface's role away from `dhcp_wan` could release dhclient without holding the same per-interface lock the renew path uses.
- The "flash ISO to a different disk" admin feature now triggers a best-effort USB rescan before listing disks, so a drive plugged in after boot is more likely to show up without a reboot. A live investigation on an actual test router found the existing disk-listing code already parses real `lsblk` output correctly -- the reported symptom on that unit turned out to be a hardware/connection issue, not a software defect; the rescan is a genuine defensive improvement, not a fix for a confirmed bug.
- Investigated a reported "DNS Proxy settings don't save" issue; could not reproduce a backend defect after tracing the full save path and testing a live create/update/read round trip directly against a running test router (everything persisted correctly). Added a permanent regression test as a guard.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `fc88f8738e70ffea24df8f3e4400cea251bf14ba`.

## v0.4.52 - 2026-08-06

Fix "flash ISO to a different disk" never listing any disks.

- The disk picker combobox always showed empty on every real router: the code expected disk sizes from `lsblk` in a text format, but real `lsblk` output uses a plain number, so reading the disk list always failed silently. The router's own disk and any other attached disks now show up correctly, with size, model, and mounted status.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `49ced324`.

## v0.4.51 - 2026-08-06

Fix the database-wedging bug for real this time (context cancellation).

- The previous release's fix addressed one code smell but the underlying database lockup ("cannot start a transaction within a transaction", every save silently failing) recurred, this time confirmed live while setting an interface's role (e.g. to dhcp_wan). Root cause: several database write operations -- live-poll's interface sync (running every ~6 seconds), the role-change reconcile behind setting dhcp_wan/pppoe_wan/etc., the "Detect" button, and VLAN creation -- used the browser request's own context for a multi-statement transaction. If that request got canceled mid-flight (tab switch, navigation, an overlapping poll), the transaction could be abandoned in a way that left the agent's single database connection stuck for every future request.
- These operations now use an internal context that can't be canceled by the browser, matching a defensive pattern already used elsewhere in the same code for the same reason.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `1665bf74`.

## v0.4.50 - 2026-08-03

Fix a leaked-transaction code smell that could wedge the database.

- Fixed a bug in `ReconcileEgressForInterfaceRole`'s unknown-interface path: it returned the result of `tx.Commit()` directly while leaving the function's internal error state as "not found", which made the deferred safety-net rollback fire after the commit had already run. Found live: the agent's single database connection got stuck ("cannot start a transaction within a transaction"), and every save (clients, NAT ports, everything) silently failed until the agent was restarted. Fixed to route the commit result correctly, and added a regression test.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `63462bf4`.

## v0.4.49 - 2026-08-01

Fix deleted client's DHCP lease resurrecting it as a new client.

- Fixed a real bug found live: deleting a client only ever removed its DB row -- the underlying dnsmasq DHCP lease for that IP/MAC was never touched, so it stayed on disk. The next periodic lease-import pass saw the still-present lease and silently recreated the exact same device as a brand-new client, even though it was no longer reachable.
- Client delete now stops dnsmasq, prunes that client's lease line, and restarts dnsmasq, the same ordering already used for static-IP reservation changes.
- Bundled DaoMai router agent/web UI from `daomai-router-minios` commit `bd238659`.

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
