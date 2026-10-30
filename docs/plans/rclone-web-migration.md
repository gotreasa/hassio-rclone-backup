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
