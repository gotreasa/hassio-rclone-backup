# Documentation review: 2026-11-10

Adversarial review of the user-facing documentation on `feat-rclone-web-poc` (POC `3.5.0-poc.26`,
commit `e36c026`, plus the uncommitted Settings-row edit in `rclone_backup/DOCS.md`). The question
was: does everything a user reads make sense, and is it true?

Docs in scope: `README.md` (`rclone_backup/README.md` is identical), `rclone_backup/DOCS.md`,
`rclone_backup/translations/en.yaml`. Every claim was checked against `config.yaml`, `apparmor.txt`,
`rootfs/` and `scheduler/`. The earlier review, `docs-review-2026-11-06.md`, is the baseline: findings
it raised that are still open are marked **(still open)**, not reported as new.

**UNREVIEWED: one reviewer (Koru), inline.** No paired reviewer ran, and nothing ran on a live Home
Assistant instance. Items marked *verified* were checked against the source; *I think* items were
not.

## Status of fixes

| #   | Finding                                       | Status                                         |
| --- | --------------------------------------------- | ---------------------------------------------- |
| N1  | Settings changes do not reach the jobs        | Fixed (Settings section)                       |
| N2  | 3.5.0 note warns of a loss that cannot happen | Fixed (note rewritten)                         |
| N7  | README pairs `sync` with a `delete` job       | Fixed (prune removed; `copy` + `delete` in DOCS) |
| N8  | Explorer folder table incomplete              | Fixed (`/config`, `/tmp`, `/data` + warning)   |
| C1  | Setup examples' `include` filter              | Fixed (filter removed; note under `include`)   |
| C2  | Two remote path styles                        | Fixed (`remote:path` everywhere; SFTP note)    |
| Rest | See below                                    | Open                                           |

## Wrong or misleading (verified)

**N1. Settings changes do not reach the jobs.** `DOCS.md:220-221` says Settings "lets you change
global rclone options". Those options go to the `rclone rcd` process behind the Web UI
(`services.d/rclone/run:10`). Each job starts its own `rclone` process (`scheduler/jobs.go:100`), so
a bandwidth limit or other option set in Settings has no effect on a job. They are also lost at
restart. A user who sets `--bwlimit` in Settings to protect their upload will not get it.
Fix: say that Settings affects the Web UI only, and point to `flags` for jobs.

**N2. The 3.5.0 "default job is gone" note warns of a loss that cannot happen.** `DOCS.md:41-43`.
The old default job on `main` had `dry_run: true` and a `google:` destination
(`git show main:rclone_backup/config.yaml`). A user who never saved the Configuration tab therefore
never had a job that copied anything: changing `dry_run` needs a save. Users who did save keep their
job. Also, "Add it again in step 3" points at a different job: step 3 adds `include: DailyBackup*`,
the old default had `include: []`.
Fix: delete the note, or reduce it to "the app no longer ships an example job".

**N3. Remotes and Settings edits when `rclone_config` is set.** With `rclone_config` set, rclone uses
`/root/.config/rclone/rclone.conf`, not `config_path` (`init.sh:9-12`), and rewrites it at every
start. `DOCS.md:161-164` and `:275` say "the file" / "the rclone config file" without naming it, and
the Settings row and section (`:119`, `:218-221`) do not warn that a Settings save is also lost at
restart in this case.
Fix: name the path once, and add the warning to the Settings section.

**N4. An invalid schedule stops the whole app, not "all jobs".** `DOCS.md:357`. A schedule error is
`Fatalln` (`main.go:146-148`) and the scheduler's `finish` script halts the container
(`services.d/scheduler/finish:14`), so the Web UI goes too. **(still open)**

**N5. `@every` does not count from app start.** `DOCS.md:351`. `scheduler.Start()` runs after all
start-up jobs finish (`main.go:154-161`). **(still open)**

**N6. `run_once` also takes down the Web UI.** `DOCS.md:260-262`. After the jobs run, `main` returns
and `finish` halts the app (`finish:14`). A user who enables it loses the Web UI with no warning.
**(still open)**

**N7. The README example contradicts itself.** `README.md:50-76` pairs a daily `sync` to
`hassbackup:` with a 14-day `delete` on the same remote. `sync` makes the remote match Home Assistant,
so either the prune never matches anything (HA keeps under 14 days) or it deletes files that the next
`sync` uploads again (HA keeps more). `README.md:76` itself says retention belongs in HA when you use
`sync`. Fix: make the first job `copy`, or drop the prune job from this example.

**N8. The Explorer folder table is not the full list.** `DOCS.md:125-135` says AppArmor "allows only
these folders". The profile also allows `/config` (this app's own folder, rw), `/tmp` (rw) and
`/data` (read) (`apparmor.txt`, rclone sub-profile). `/data/options.json` holds the `rclone_config`
option, so Explorer can show account tokens in clear text. Fix: add the three rows; consider a
one-line warning on `/data`. **(partly raised before, still open)**

**N9. "Look in the logs" for AppArmor denials does not say which log.** `DOCS.md:139-140`. Denials go
to the host's kernel log, not the app's **Log** tab. **(still open)**

**N10. The README installs upstream, not this build.** The badges and repository URL point at
`jcwillox/hassio-rclone-backup` (`README.md:16-23`, `config.yaml:13`) while the image is
`ghcr.io/gotreasa/...` (`config.yaml:61`). A POC tester following the README gets upstream 3.4.1,
whose docs do not match. Correct if this text is for the upstream PR; wrong for testers of the fork.
Needs a decision. The menu path **Supervisor** → **Add-on store** is also out of date (I think the
current path is **Settings** → **Apps**; check before changing).

## Will confuse a reader

**C1. The include filter in both quickstart examples will probably match nothing.**
`DOCS.md:25-26` and `:79-80` use `include: DailyBackup*`. That name comes from the Auto Backup
integration (`README.md:4`); `DOCS.md` never says so. I think HA's own automatic backups are named
"Automatic backup &lt;version&gt;", so a new user's first job copies nothing and still reports success.
Fix: drop `include` from the quickstart, or explain where the name comes from and point at the preview
job.

**C2. Two path styles for the same destination.** `google:/Backup/Home Assistant` (`DOCS.md:24`)
versus `mydrive:Backup/Home Assistant` (`:78`). On some backends these are different places (SFTP:
leading `/` is absolute, without it the path is under the home folder). Pick one, and say why if it
matters. **(still open)**

**C3. The `dry_run` step tells you to do two things.** `DOCS.md:84-86` says "Set `dry_run` to
`false`", then "You can keep `true` for the first run". The opening example (`:29`) also shows
`false`. Reorder: keep `true`, restart, read the log, then set `false`. **(still open, P3)**

**C4. Snippets that fail when pasted.** The prune example (`DOCS.md:379-387`) is a bare list item with
no `jobs:` key. The preview job (`:399-408`) has no schedule, so it runs at every start until removed;
say to remove it afterwards.

**C5. Missing reference entries.** `log_level` is in the schema (`config.yaml:60`) and not documented;
only `debug` changes anything (`log.go:19`). `command` is the only required job key and the docs do
not say so. **(still open)**

**C6. The slug rule is incomplete.** `DOCS.md:395` says special characters become `_`; the slug
library turns others into `-` (`main.go:79-89` lists only the `_` cases). Point readers at the preview
job instead of describing the rule. **(still open)**

**C7. "Restoring backups through the WebUI".** `DOCS.md:297` now reads as rclone-web; it means HA's
Backups page. Also drop "apparently". **(still open)**

**C8. Only Samba is proven for sharing a mount.** `DOCS.md:211-213` lists Samba, NFS Server and FTP
as equals. Only Samba was confirmed live (`docs/plans/rclone-web-migration.md:220-221`). Mark the
other two as untested, or test them.

**C9. Leftovers.** The "Configuring Rclone Remotes" stub (`DOCS.md:424-428`) repeats **Remotes**.
Terms are mixed: app / addon / add-on, Web UI / WebUI (`DOCS.md:262, 273, 297, 341, 438`, most of
`README.md`). "Backup your" should be "Back up your" (`DOCS.md:3`, `README.md:2`). README provider
list names dead backends ("Amazon Drive (See note)" with no note, "Uptobox") and says `googledrive`
where the type is `drive` (`README.md:27`). **(mostly still open)**

**C10. "3.5.0 and later" appears three times** (`DOCS.md:41, 389`, and the version gating). If the
`feat!` change releases as 4.0.0 these become wrong. Decide at release time.

## Gaps a new user will hit

- No "check your first run" and no "recover a file" section. **(still open, P1 and P2)**
- Help text exists for 3 options only (`translations/en.yaml`). **(still open, P5)**
- A failed job on `/backup` leaves the backups renamed: `RunJob` returns at `jobs.go:105-110` before
  `undoRename` at `:114`. This is the open code bug B1; until it is fixed, Troubleshooting should say
  what the user will see.

## Checked and correct

The uncommitted Settings row ("Saving the config contents creates the file if it does not exist")
matches `rclone-web/settings-save.patch`, whose e2e test saves to a missing file. Also correct: a job
naming an unknown remote stops the app and the Web UI (`main.go:122-125`, `finish:14`); a missing
`rclone.conf` only warns (`main.go:103-104`); the four `delete` rules (`main.go:206-224`); `source`
and `destination` override the plural forms (`main.go:116-121`); `dry_run` pre-filled `true`
(`config.yaml:29`, also on `main`); the `/dev/fuse` and `SYS_ADMIN` set-up for mounts; Samba sharing
of a mount under `/share` or `/media` (confirmed live, see C8).
