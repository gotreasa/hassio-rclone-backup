# Documentation review: 2026-11-06

End-to-end review of the user-facing documentation on the `worktree-rclone-web-migration` branch
(POC `3.5.0-poc.22`, commit `c589f40`), in two passes:

1. **Accuracy audit.** Every claim in the docs checked against the code and add-on config.
2. **Newcomer pedagogy review** by Sage (`gsmith-pedagogy-reviewer`, the gSmith deep reviewer). Sage
   is built for Galvanize Learn modules, so the Learn DSL, UID and CI lint checks were switched off.
   It judged flow, pitch, gaps and correctness for a reader who runs Home Assistant but has never used
   rclone.

Docs in scope: `README.md` (`rclone_backup/README.md` is a symlink to it), `rclone_backup/DOCS.md`,
`rclone_backup/CHANGELOG.md` and `rclone_backup/translations/en.yaml`.

**Limits.** Both passes read the code; nothing ran on a live Home Assistant instance. Koru
re-checked the findings marked *verified* against the source. The rest are the reviewers' findings,
not yet re-checked.

## Status of fixes

| #   | Finding                                              | Status                         |
| --- | ---------------------------------------------------- | ------------------------------ |
| W1  | Preview job example stops the app                    | Fixed                          |
| W2  | `/config` described as the Home Assistant folder     | Fixed (docs only, see note)    |
| W3  | `no_events` documented but not in the schema         | Fixed (added to the schema)    |
| W4  | Job-level `extra_flags` described as global          | Fixed                          |
| W5  | Web UI remotes lost when `rclone_config` is set      | Fixed                          |
| B1  | Backups stay renamed after a failed job (code bug)   | Open                           |
| P1  | No "recover a file" section                          | Open                           |
| P2  | No "check your first run" section                    | Open                           |
| P3  | `sync` default is destructive, no warning at use     | Open                           |
| P4  | Laptop-first vs Web UI-first setup message           | Open                           |
| P5  | HA options form has help text for 2 fields only      | Open                           |

Note on W2: `init.sh:6` still runs `ln -s /homeassistant /config`. Because Supervisor already mounts
the app's own folder at `/config` (`addon_config:rw`, `config.yaml:19`), that line creates
`/config/homeassistant` instead of the alias its comment promises. This is also true on `main`. The
docs now describe the real behaviour; the script is unchanged.

## Wrong: verified against the source

**W1. The slug preview example stops the app.** `DOCS.md` showed a preview job with
`destination: /tmp/preview`. Every local destination is checked at start-up and a missing one is
fatal (`scheduler/main.go:216-220`). When the scheduler exits, the whole app halts
(`rootfs/etc/services.d/scheduler/finish:14`). Nothing creates `/tmp/preview`.
Fix: use `/tmp`, which always exists.

**W2. `/config` is the app's own folder.** `DOCS.md` said "`/config` is another name for
`/homeassistant`, so older paths such as `/config/rclone.conf` still work". `README.md` and the
`sources` description also offered `/config` as the Home Assistant configuration. With
`addon_config:rw` mapped, `/config` is `/addon_configs/<slug>`.
Fix: say what `/config` is, and use `/homeassistant` in the examples.

**W3. `no_events` cannot be set.** The code reads it (`scheduler/main.go:43`, `events.go`), but the
`config.yaml` schema did not declare it, so Supervisor would not accept it. It is also missing on
`main`. Fix: add `no_events: bool?` to the schema.

**W4. Job-level `extra_flags`.** The job option was described as "applied globally to all jobs".
Job flags go only on that job's command (`scheduler/jobs.go:83`); the global ones are at
`jobs.go:81`. Fix: separate wording for the job option.

**W5. Where Web UI remotes are saved.** The docs said the Web UI writes to
`/homeassistant/rclone.conf`. rclone uses `RCLONE_CONFIG`, set from `config_path`, or to
`/root/.config/rclone/rclone.conf` when `rclone_config` is set (`init.sh:9-17`). That file is
rewritten from the option at every start (`init.sh:12`), so remotes added in the Web UI are lost.
Fix: name the file the app uses and warn about `rclone_config`.

## Code bug found during the audit

**B1. Backups stay renamed after a failed job.** If rclone fails, `RunJob` returns before
`undoRename` runs (`scheduler/jobs.go:114-125`), so the backups in `/backup` keep their friendly,
slugified names. This needs a code fix with a regression test, in its own `fix:` commit.

## Newcomer review (Sage)

**Verdict: NEEDS_REVISION.** No wrong content was found that would earn REJECTED, but two of the
five learner goals cannot be done from the docs alone.

### Learner journey

| Step                         | From the docs alone? | Why                                                                                   |
| ---------------------------- | -------------------- | ------------------------------------------------------------------------------------- |
| 1. Install the app           | Partly               | Badges are clear; the "read Before you start" warning is buried in a paragraph.      |
| 2. Connect a cloud remote    | Barely               | Needs rclone on a second computer; "remote" is only defined in brackets.             |
| 3. Set up a scheduled job    | Partly               | Good YAML and schedule table; no word on the YAML editor; `dry_run` advice conflicts. |
| 4. Confirm it worked         | No                   | Default runs at 04:10; no "what success looks like" or "browse the destination".     |
| 5. Recover a file            | No                   | The only mention of restore is a side note under `no_unrename`.                      |

### Top 5 changes by learner impact

1. **P1. Add "Recover a file".** Copy a backup back to `/backup` and reload HA's backups; get one
   file out through Explorer or a mount; say which filenames to expect (`no_rename`, slugify); say
   that a crypt remote needs its passwords to recover.
2. **P2. Add "Check your first run".** Clear `schedule` so the job runs at start-up, show a success
   log line, browse the destination, and name `rclone_backup.job_successful`.
3. **P3. One quickstart path with a `sync` warning.** The shipped job is `sync` to
   `google:/Backup/Home Assistant`; on a folder with other files, `dry_run: false` deletes them.
   Write the dry run, read log, real run sequence as one path (`DOCS.md` step 3 offers both).
4. **P4. Resolve the bootstrap message.** "Create the config on a laptop first" competes with "add
   remotes in the Web UI", but the Web UI is unreachable until every job's remote resolves. Put a
   callout under Installation and define "remote" at first use.
5. **P5. Fill in `translations/en.yaml`.** Only `jobs`, `command` and `schedule` have text. Start
   with `dry_run`, `destination`, `include`, `config_path` and `rclone_config`.

### Other suggestions from Sage

- Move Mounts and the AppArmor detail below the Configuration reference; keep one goal-ordered
  spine: Install, Remote, Job, Check, Recover.
- Use one path form in the worked example (`google:/Backup/...` vs `mydrive:Backup/...`).
- Lead the README with a single plain remote; keep crypt as an advanced example and say to create
  it with `rclone config`, not by hand.
- Pick one of "add-on", "addon" and "app".
- Troubleshooting: add "app stops: remote not found", "invalid schedule" and "Explorer permission
  error" before the OneDrive Personal Vault entry.
- Remove or merge the "Configuring Rclone Remotes" stub.

## Smaller accuracy findings (not re-checked by Koru)

- iCloud is given as a browser-login provider; rclone's `iclouddrive` asks for an Apple ID and
  password.
- The slug rule is incomplete: some characters become `-`, not `_` (`scheduler/main.go:81-89`, plus
  gosimple/slug).
- `command` is the only required job key; the docs do not say so (`config.yaml:43`).
- `log_level` is undocumented; only `debug` changes anything (`log.go:18-21`).
- `run_once` also stops the Web UI (`services.d/scheduler/finish`).
- An invalid schedule stops the whole app, not only the jobs (`main.go:146-148`).
- `@every` counts from when the start-up jobs finish, not from app start (`main.go:154-161`).
- "The file must exist before the app starts": a missing file only warns (`main.go:103-104`); the
  real stop is a job naming an unknown remote.
- The AppArmor folder table omits `/config` and `/tmp`, and does not say `/data` is readable.
  `/data/options.json` holds any `rclone_config` secrets; consider this for `docs/security/`.
- `DOCS.md` says "3.5.0 and later" for the command limit; the `feat!` commit will likely release as
  4.0.0.
- `CHANGELOG.md` tops out at 3.4.1 while `config.yaml` is `3.5.0-poc.22` (expected for a POC).
- README: the menu path still says **Supervisor** → **Add-on store**; the provider list names
  backends rclone no longer has ("Amazon Drive (See note)", "Uptobox"); the badges point at upstream,
  so on the fork they install upstream 3.4.1; "`googledrive`" should be the type name `drive`.
- AppArmor denials appear in the Host log, not the app's Log tab.
- "Restoring backups through the WebUI" now reads as rclone-web; it means HA's Backups page.
- The OAuth token can be a base64 blob as well as `{...}` JSON (`oauth-headless.patch`).
- Say where to find the app's rclone version (Log tab; `Dockerfile:66` pins 1.74.3).
- `dry_run` is pre-filled as `true`; if the key is removed the code default is `false`.
- Flag values: the schema regex accepts only quoted strings, `None`, `False` and `True`, so numbers
  probably need quotes (not checked against Supervisor's coercion).

## Verified as correct

Option names and types against the schema; the copy, sync and move limit; `source` and
`destination` overriding the plural forms; flag name conversion; the schedule table (five fields,
descriptors, `@every`, `CRON_TZ=`, empty runs at start-up, YAML quoting); start-up job and remote
checks; default `config_path` and `rclone_config` precedence; `no_rename`, `no_unrename` and
`no_slugify`; event names and attributes; the no-login ingress set-up; the Web UI tab list with
Serves and Download removed; the browser-login flow; all 17 external links (HTTP 200); the README and
DOCS example YAML against the schema.

Not verifiable without upstream rclone-web 1.2.0 source or a live instance: the Dashboard contents,
some Mounts labels, what the Settings tab shows, and how Supervisor treats a saved job with an
invalid `command`.
