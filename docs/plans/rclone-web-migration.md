# Plan: migrate the web GUI from rclone-webui-react to rclone-web

## Goal

`rclone/rclone-webui-react` is archived. Upstream recommends `rclone/rclone-web`. Bundle rclone-web
in the add-on image instead, and prove it works behind Home Assistant ingress.

## Sizing

Size S (nWave `nw-auto` §6): a dependency swap against a settled upstream decision. Direct work plus
an independent reviewer, not the full DES step loop. Research by `nw-researcher`.

## Steps

- [x] Read the current wiring: `rclone_backup/Dockerfile:41-58`, `rootfs/etc/services.d/rclone/run`.
- [x] Build the baseline image (aarch64) locally: builds.
- [x] Research rclone-web: release asset, layout, sub-path support, API discovery (`nw-researcher`).
- [x] Patch rclone-web for sub-path serving (`rclone_backup/rclone-web/base-path.patch`).
- [x] Change the Dockerfile, the run script, the Renovate annotation and the docs.
- [x] Build the new image (aarch64) and test it behind a simulated ingress proxy: 7/7 pass.
- [ ] Cross-build armhf locally: blocked, see Blockers. CI on the PR builds all five arches.
- [x] Independent review of the diff (`nw-platform-architect-reviewer`): conditionally approved, no
  blockers. See Review below.
- [x] Fix review findings 1, 2 and 4. Final patch: biome, `tsc -b` and `vite build` pass locally.
- [ ] Rebuild the image with the final patch and rerun the browser check (now 9 steps, including an
  ingress token change). Blocked on Docker, see Blockers.
- [x] Commit on the branch: `0b86699` (2026-10-30, signed after a 1Password approval). Push and
  open a draft PR only on Gearoid's go.
- [ ] Optional, Gearoid's call: offer the base-path patch upstream to `rclone/rclone-web`.

## Proof of concept on a fork (Gearoid, 2026-10-30)

A POC only: nothing goes to `jcwillox/hassio-rclone-backup` (no push access there anyway).
No Jira item, by Gearoid's choice.

- [x] Fork: https://github.com/gotreasa/hassio-rclone-backup (public). Fork setting changed:
  default workflow token `read` to `write`, because `deploy.yaml` pushes to GHCR and declares no
  `permissions:` block.
- [x] Fork-only commit `chore(poc): ...`: `config.yaml` image `ghcr.io/gotreasa/...`, version
  `3.5.0-poc.1`, POC name; `repository.json` name and URL. **Drop this commit before any upstream PR.**
- [x] Push to the fork as `feat-rclone-web-poc`. Gearoid enabled the fork's workflows by hand (GitHub
  keeps a new fork's workflows off until the owner confirms in the Actions tab).
- [x] CI on `c2535f1` (manual dispatch): lint plus all five arch builds pass, which closes the
  "cross-arch builds unverified" gap. https://github.com/gotreasa/hassio-rclone-backup/actions/runs/40987625260
- [x] Prerelease `3.5.0-poc.1` on `c2535f1`; Deploy pushed all five images.
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/40988170855
- [x] Images are public with no manual step: an anonymous manifest pull of
  `ghcr.io/gotreasa/hassio-rclone-backup/<arch>:3.5.0-poc.1` returns HTTP 200 for all five arches.
- [x] Ingress hardening (Gearoid's go, 2026-10-30): HA's ingress guide says "Only accept connections
  from 172.30.32.2. Deny everything else." rclone now listens on `127.0.0.1:5572`; nginx on 8099
  allows only `172.30.32.2` and proxies to it; `--rc-allow-origin` (a CORS header, not access
  control) dropped. Tested on an HA-shaped network (`172.30.32.0/23`, proxy at `172.30.32.2`, a
  second container at `172.30.33.10`): browser check 9/9 through the proxy; the second container
  gets 403 on 8099 and a refused connection on 5572. Negative control: the `3.5.0-poc.1` image
  answers that container with 200. This run also verifies the final base-path patch. POC version
  bumped to `3.5.0-poc.2`.
- [x] `3.5.0-poc.2` released on the fork from `6d3b843`: CI (all five arches)
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41046546465 and Deploy
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41046817141 pass; all five images
  pull anonymously.
- [x] Confine the Web GUI (Gearoid's go, 2026-10-30: `/homeassistant` and `/backup` only).
  rclone has no flag for this (checked `rclone help flags` in 1.74.3), and a UI-only limit is
  bypassed by `config/create` alias remotes or `core/command`, so it is done at the OS level: the
  GUI's rclone runs under `unshare` with private mount and PID namespaces, the other HA folders
  lazily unmounted, a tmpfs over `/run/s6/container_environment`, and `SUPERVISOR_TOKEN` removed
  from its environment. The scheduler stays in the normal namespace. Needs `util-linux-misc`
  (busybox `unshare` has no `--kill-child`).
  - Found while testing: util-linux `unshare --fork` ignores SIGTERM, so s6 could not restart or
    stop the GUI (an early restart check passed vacuously; replaced by one that requires a new
    PID). Fixed with `down-signal` SIGHUP and `--kill-child=SIGTERM`.
  - Found while testing, pre-existing: `rclone/finish` treated exit 143 (SIGTERM, sent by s6 on
    every stop or restart) as a crash, so a GUI restart halted the whole add-on. Fixed by treating
    143 as a normal stop.
  - Rig (full s6 `/init`, fake Supervisor, HA folders mounted with markers, `SYS_ADMIN`): 28/28
    isolation checks, including bypass attempts (alias remote, `/proc/1/root`, s6 env store,
    `core/command cat` of environ), a token sweep of the GUI's namespaces, the scheduler still
    seeing every folder, and a real restart; browser check 9/9; `docker stop` 3 s, exit 0.
  - POC version bumped to `3.5.0-poc.3`.
- [x] `3.5.0-poc.3` released on the fork from `35e5d87`: CI (all five arches)
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41051713708 and Deploy
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41051961397 pass; all five images
  pull anonymously.
- [x] **Confinement reverted** (Gearoid, 2026-10-30): "The changes for poc.3 didn't help and can be
  reverted." Reasoning agreed: only HA admins reach the GUI after `poc.2`, and they can already
  read every file through other add-ons, so confining the Explorer guards against trusted users.
  `git revert` of `1e22bb6` as `9b01107` (no history rewrite). Kept `5ea7874` (the `finish` fix),
  because it fixes a pre-existing bug: a GUI service restart halted the add-on. Checked on the
  reverted build in the full s6 rig: other add-on 403, rclone on loopback only, a real GUI restart
  (new PID) with the add-on still running and no crash warning, browser check 9/9, `docker stop`
  exit 0 in 4 s. POC version bumped to `3.5.0-poc.4`.
- [x] `3.5.0-poc.4` released on the fork from `4355687`. CI passed (all five arches):
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41058081726. Deploy attempt 1
  failed at job setup on armhf ("Unable to resolve action `actions/cache@v5.1.0`, unable to find
  version"), and fail-fast cancelled the rest. Classified as a transient GitHub action-resolution
  failure: both pinned releases exist (`actions/cache` v5.1.0 published 2026-06-26,
  `docker/login-action` v4.2.0 published 2026-05-22), the other jobs in the same run resolved
  them, and the workflow was unchanged since three green Deploy runs. Rerun of the failed and
  cancelled jobs (attempt 2) passed for all five:
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41058362157. All five images pull
  anonymously.
- [x] Custom AppArmor profile (Gearoid's go, 2026-10-30): `rclone_backup/apparmor.txt`, and
  `apparmor: true` on `config.yaml:73` (was `false`, set in `1362bde` for FUSE mounts). Built on the
  S6 overlay example in HA's app security guide. rclone (Web GUI server and scheduled jobs) runs in
  a stricter child profile: the mapped folders (`ssl` and `data` read-only), FUSE mount and
  umount, network, its own config and caches, and no other programs. Expected rating 7/8
  (5 + 2 ingress + 1 custom profile − 1 SYS_ADMIN); badge green.
  - **Ships in `complain` mode**, per HA's method: AppArmor logs what it would block instead of
    blocking. The badge is green in complain mode too, so it does not prove enforcement.
  - Not testable here: Docker Desktop's kernel has no AppArmor (`docker info` security options:
    seccomp, cgroupns). `apparmor_parser` 3.1.7 compiles it (`--skip-kernel-load`); a broken copy
    is rejected, so the check is real.
  - POC version bumped to `3.5.0-poc.5`.
  - CI's add-on linter rejected `apparmor: true` ("'apparmor' should be removed, it uses a default
    value"). HA's default is `true`, so Gearoid chose to remove the line: same behaviour, the
    profile is still loaded. Lint warnings about armhf, armv7 and i386 (deprecated since HA
    2025.12) are pre-existing and out of scope.
- [x] `3.5.0-poc.5` released on the fork from `a52f19e`: CI (lint and all five arches)
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41063182842 and Deploy
  https://github.com/gotreasa/hassio-rclone-backup/actions/runs/41063489997 pass; all five images
  pull anonymously; the branch serves `apparmor.txt` with `complain` on both profiles.
- [x] `poc.5` installed on Gearoid's instance. The first attempts (13:32:09, 13:32:40) failed with
  "An unknown error occurred with app e2fe72d5_rclone_backup": HA offers the update as soon as the
  branch's `config.yaml` changes, but the aarch64 image was only pushed at 13:33:09. Gearoid
  confirmed the timing was the cause. **Process: wait for Deploy to finish before updating.**
- [x] Question answered (2026-10-30): what removing `privileged: SYS_ADMIN` would do. Only
  `rclone mount` breaks (Web GUI Mounts tab, `mount/mount`, any job with `command: mount`):
  with SYS_ADMIN a FUSE mount works; without it rclone's mount daemon exits with code 1 and no
  mount exists. Everything else passes without it in the full s6 rig (403 for other add-ons, real
  GUI restart, browser 9/9, clean stop). Rating would go from 7 to 8. Not changed; Gearoid's call.
- [x] Experiment `poc.6` (Gearoid's ask, 2026-10-30): `poc.5` minus `privileged: SYS_ADMIN` only
  (`ce81918`; `/dev/fuse` and the profile unchanged), to check on real HA that only mounts break.
  Released from a staging branch first, and `feat-rclone-web-poc` moved only after Deploy, so HA
  could not offer the update before the images existed. **Result on Gearoid's instance: the mount
  works on `poc.5` and fails on `poc.6`.** Decision: keep SYS_ADMIN; `ce81918` reverted as
  `a153263`.
- [x] Deprecated map types (Supervisor log, 2026-10-30: "uses legacy map type 'addons'; use
  'local_apps' instead", and the same for `addon_config` and `all_addon_configs`). Deprecated in
  Supervisor 2026.07 (`supervisor/apps/validate.py:263`). The new types mount elsewhere by default
  (`local_apps` at `/local_apps`, `all_app_configs` at `/app_configs`; HA configuration reference),
  so `1fd9570` uses the long form with `path:` to keep `/addons` and `/addon_configs`
  (`read_only: false`, as the default is true). `app_config` already mounts at `/config`.
  - Upstream note: the new types need Supervisor 2026.07 or later; older Supervisors would
    probably reject the config. **Researched 2026-10-30 (Gearoid: "a massive problem"):**
    - The new types first exist in **Supervisor 2026.07.1** (2026-07-08). `MappingType` read at
      every release tag from 2025.07.1 to 2026.10.5: none of the three in 2026.07.0 or earlier,
      all three from 2026.07.1 (`supervisor/apps/const.py`, earlier `supervisor/addons/const.py`).
    - On 2026.07.0 or earlier, `vol.Coerce(MappingType)` (`apps/validate.py:448`) rejects
      `local_apps`, and the store logs "Can't read config.yaml" and skips the app
      (`store/data.py:200-205`): **it disappears from the store, so no install and no update.**
      What happens to an already-installed copy is not verified.
    - The Supervisor updates itself by default (`supervisor/validate.py:196`
      `ATTR_AUTO_UPDATE default=True`; `supervisor.py:271-289`), so exposure is users who
      turned auto-update off, or whose Supervisor cannot update. Not measured.
    - The old names have **no removal date**: HA's reference says "Older names are still
      accepted, with a warning", and `apps/validate.py` only logs for them. They work on every
      Supervisor, old and new.
    - A gate does not help: `homeassistant:` sets a minimum Core version, not Supervisor, and an
      old Supervisor fails while parsing, before any gate applies.
    - **Decision (Gearoid, 2026-10-30): back to the old names.** Reverted `087c4f0` and `1fd9570`
      as `a5085e4` and `28be77e`; `map` is identical to `main` again. Kept the linter bump
      `6b79ef6` (v2.21.1 accepts the old names; both linter schemas validate the config).
      Cost: three warnings in the Supervisor log. Revisit when HA announces a removal date for
      the old names. POC version bumped to `3.5.0-poc.8`.
  - POC version bumped to `3.5.0-poc.7`. Gearoid to check the Supervisor log shows no legacy map
    warnings, and that `/addons` and `/addon_configs` still show their contents.
  - CI lint failed on the first attempt (run 41083760301: "is not valid under any of the given
    schemas"). Two causes, both checked against the linter's `src/config.schema.json`:
    `frenck/action-addon-linter@v2.21.0` does not know the new types (v2.21.1 does), and its
    schema accepts a `map` of only strings or only objects, never a mix. Fixed by bumping the
    linter to v2.21.1 (`6b79ef6`) and writing every entry in long form (`087c4f0`). Validated
    locally with `jsonschema`: v2.21.1 VALID, v2.21.0 still invalid.
- [ ] Gearoid: run `poc.5` and use every feature (Web GUI, a job, a mount), then read the audit log
  for `apparmor="ALLOWED"` entries naming `rclone_backup`. Add what is missing, then remove
  `complain` in `poc.6`.
- [ ] Gearoid: add `https://github.com/gotreasa/hassio-rclone-backup#feat-rclone-web-poc` as an
  add-on repository in HA, install, open the panel.

## Findings

- **The stock rclone-web build cannot run under ingress.** Verified in the 1.2.0 `dist.zip`:
  `index.html` loads `/assets/index-*.js` and `/icon.svg` by absolute path; the app uses
  `createBrowserRouter` with no `basename`; `src/rclone/client.ts` redirects to `/login`; the rc URL
  defaults to `''` in `src/lib/store.ts`, so the first visit shows a login form with no URL.
  The old rclone-webui-react build used relative paths (`./static/...`), which is why it worked.
- **Fix: build from source at a pinned tag with a small patch.** Vite `base: './'`; router basename,
  icon paths and login redirects derived at runtime from the bundle's own URL (`src/lib/base.ts`);
  the rc URL defaults to the serving origin and path. Biome clean, `tsc -b && vite build` clean.
- **The node stage uses `--platform=$BUILDPLATFORM`.** The output is plain JS, and official `node`
  images have no armhf or i386 builds, so building per target arch would fail.
- **`rc/noopauth` works under `--rc-no-auth`** on rclone 1.74.3 (`{}`, HTTP 200).
- **Ingress test** (nginx at `/api/hassio_ingress/TESTTOKEN/`, prefix stripped, add-on's own run
  command, headless Chromium): lands on the dashboard with no login, rc URL stored as the ingress
  base, icons load, nav links are prefixed, `/remotes` lists the configured remote, reload at the
  panel root works, no console errors, no HTTP errors, no request escapes the prefix.
  **Negative control:** the stock 1.2.0 zip behind the same rig renders a blank page and fails.
- **Known limit, unchanged in practice:** `rclone rcd --rc-files` has no SPA fallback, so a browser
  reload on a deep link (for example `.../remotes`) returns 404. HA always opens the panel at its
  root, so this does not bite through the HA UI.
- **Not changed:** `example.png` still shows the old UI. A new screenshot is a separate, optional step.
- `rclone gui` (rclone 1.74) embeds rclone-web 1.1.8 but listens on two random localhost ports, so
  it does not fit single-port ingress; `--rc-web-gui` still fetches the archived rclone-webui-react.

## Review (2026-10-29, `nw-platform-architect-reviewer`, conditionally approved)

Each finding was re-checked against source before acting on it.

1. **Fixed.** The persisted rc URL would go stale if the ingress token changed (add-on reinstall):
   `partialize` in `src/lib/store.ts` stored the absolute same-origin URL, which then overrode the
   default. The patch now leaves `url` out of storage when it equals the same-origin default.
2. **Fixed.** A Renovate minor bump could automerge a GUI broken under ingress (`:automergeMinor`,
   CI only builds). The Dockerfile now applies the patch with `--fuzz=0` and fails the build if
   `dist/index.html` loads anything by absolute path; `renovate.json` gives rclone-web its own
   group with automerge off.
3. Rejected in part: "busybox `patch` is already in node:alpine" is false (`command -v patch`
   printed MISSING in `node:24-alpine`), so `apk add patch` stays. No checksum on the tag tarball
   and the floating `node:24-alpine` tag match the existing Dockerfile's pattern; left as is.
4. **Fixed.** Renovate datasource is now `github-releases`, matching the other dependencies.
5. to 6. No change: index.html caching is the same exposure as before; `client.ts:170` is dead
   upstream code.
7. `example.png` still shows the old UI (unchanged, optional follow-up).

Not validated: `renovate.json` against Renovate's schema (running the validator via `npx` was
blocked by the session's safety classifier). It parses as JSON.
Reviewer's caveat, accepted: the nginx rig does not model HA auth, the frontend iframe, HTTPS or
Nabu Casa. One smoke test on a real HA instance before release.

## Blockers

- 2026-10-29: the Mac's disk reached 99% (133 MiB free) during the armhf cross-build, and the Docker
  daemon stopped answering (`500` on `_ping`). Local cross-arch builds wait for free disk space.
- 2026-10-29 23:50: disk recovered to 4 to 5 GB free, but Docker Desktop did not come back after a
  relaunch (no process, socket refused for 5 minutes). Repairing it is Gearoid's call.
