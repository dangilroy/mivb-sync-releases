# MiVB Recording Coverage Sync

`MivbSync.exe` is an unattended Windows tool that keeps ASC Neo recording coverage complete for
a Mitel MiVoice Business (MiVB) estate. Every night it reads the MiVB ESM Scheduler's "User and
Services Configuration" CSV exports from an SFTP server. It compares every extension with the
monitor points in ASC Neo, creates the ones that are missing, and reports anything it must not
change.

This repository holds the published releases only. Download the latest from
[Releases](../../releases/latest). Installed copies update themselves from here (see
[Updates](#updates)).

> **Required: the MiVB export job must be called `ascsync`.** On every controller, the ESM
> Scheduler export job's **File Name** (under Activity Parameters) must be exactly `ascsync`.
> The tool only reads exports named `ascsync_CSVExport_<controller>_<timestamp>.csv`. Any
> other name is ignored, and that controller reports NO_FILES.

- [How it works](#how-it-works)
- [What you need](#what-you-need)
- [Install](#install)
- [Set up with the setup window](#set-up-with-the-setup-window)
- [Check the installation](#check-the-installation)
- [The nightly task](#the-nightly-task)
- [First runs (pilot)](#first-runs-pilot)
- [Reading the results](#reading-the-results)
- [Updates](#updates)
- [Configuration reference](#configuration-reference)
- [Command reference](#command-reference)
- [Troubleshooting](#troubleshooting)

## How it works

```
MiVB ESM Scheduler -> CSV export on SFTP -> MivbSync.exe (nightly) -> ASC Neo (REST) -> report
```

For each configured MiVB controller, every run:

1. Picks that controller's newest export from the shared export folder.
2. Checks the export: it must be recent, new since the last run, finished writing, complete,
   and not sharply smaller than last time.
3. Reads the monitor points from ASC Neo.
4. Creates an **active monitor point** under the controller's integration for every extension
   that has none. This is the only thing it writes to Neo.
5. Flags what it must not change: monitor points Neo has but MiVB does not, extensions under the
   wrong integration, inactive monitor points.
6. Writes a report (text, HTML and JSON) and a log.

It is built to fail safe. A missing, old, unchanged, partial or much smaller export is never
read as "these extensions were removed", and nothing is deleted from Neo. (The one exception is
the optional `reassign_mode = "move"`.)

> An active monitor point does not by itself guarantee that calls are recorded. Recording still
> depends on the recording plan and licences in ASC Neo. "Covered" here means an active monitor
> point under the integration.

## What you need

- **A Windows server or VM** (the "sync host") with:
  - HTTPS to the ASC Neo server (`https://<server>/ASCWebService/rest`);
  - SFTP (port 22) to the server that receives the MiVB exports;
  - outbound HTTPS to GitHub for self-updates (see [Updates](#updates)). A proxy can be set
    with the `HTTPS_PROXY` environment variable.
- **Local administrator rights** on the sync host, to install the scheduled task. The task runs
  as SYSTEM, so no service account or stored Windows password is needed.
- **An ASC Neo account**: a superuser of the System Provider tenant. Under System Configuration >
  Tenants > System > tab "Web Service", turn on the master switch and the MonitorPoint methods
  (importing and exporting monitor points).
- **A dedicated SFTP account that can see the export folder only.** Optional cleanup deletes old
  exports, so the account must not reach anything else.
- **A MiVB ESM Scheduler export job called `ascsync` on every primary controller.** Set it up
  as follows:
  - **Form:** User and Services Configuration.
  - **Schedule:** nightly.
  - **Destination:** the one shared SFTP folder.
  - **File Name** (under Activity Parameters): exactly **`ascsync`**. This is not optional.

  The Scheduler then names the files `ascsync_CSVExport_<controller>_<YYYYMMDDHHMMSS>.csv`.
  Exports with any other File Name are never read or deleted, so other export jobs can safely
  share the folder.

## Install

1. Download `MivbSync.exe` and `config.example.toml` from the
   [latest release](../../releases/latest). Optionally check the exe against `SHA256SUMS.txt`:
   `Get-FileHash MivbSync.exe` in PowerShell.
2. Put both files in a folder, for example `C:\Program Files\MivbSync\`.
3. Create the data folder `C:\ProgramData\MivbSync` (the default `data_dir`). If you choose
   another folder, give SYSTEM Modify rights on it.

You don't need to copy `config.example.toml` to `config.toml` by hand: the setup window creates
`config.toml` from it.

## Set up with the setup window

Double-click `MivbSync.exe`, or run `MivbSync.exe setup`. Accept the UAC prompt: installing the
scheduled task needs an administrator. Each panel's header shows its state: a green tick when it
is done, amber for what is still to do, red when a test failed.

1. **Customer**: the customer name, shown in reports.
2. **ASC Neo**: server (a host name, `host:port` or a URL), username and password. Click
   **Test**. A passing test also reads the integration names from Neo for the next panel.
3. **Controllers**: one row per **primary** MiVB controller.
   - **Name** must match the controller name in the export file name exactly.
   - **Integration** is the ASC Neo integration its extensions belong to. Pick it from the list,
     or type it exactly as in Neo. Several controllers may share one integration.
   - **Element id**: leave it blank unless one controller's export contains sets homed on other
     controllers (see [Configuration reference](#configuration-reference)).
4. **SFTP server**: host, port (22), username, password and the export folder. **Browse...**
   picks the folder on the server. **Test** logs in, lists the folder and shows how many exports
   each controller has. The server's host key is remembered on the first connection; the tool
   refuses to connect if it ever changes.
5. **Save**. The whole file is checked before it is written, and the previous version is kept
   as `config.toml.bak`. Passwords go into Windows Credential Manager, never into the file. A
   stored password shows as `********`.
6. **Scheduled task**: choose the time (default 02:00), then click **Install / Update task**.
   This also copies the saved passwords to SYSTEM, which runs the task. Pick a time after the
   MiVB exports have finished, with some margin.

Every other setting (safety limits, row filters, cleanup and so on) lives in `config.toml` (see
[Configuration reference](#configuration-reference)). There is also a console version of the
setup: `MivbSync.exe setup --no-gui`.

## Check the installation

Open a command prompt in the install folder and run:

```
MivbSync.exe check-config
MivbSync.exe test-connections
```

`check-config` validates `config.toml` and lists the controllers. `test-connections` checks Neo
(reachable, login, permissions) and SFTP (login, folder, exports per controller).

To check how an export is read:

```
MivbSync.exe parse-file --controller MIVB-LON01 C:\temp\ascsync_CSVExport_MIVB-LON01_20261007163000.csv
```

This shows the columns found, the row counts, any problem rows, and which rows the filters drop.

These commands run as you, with the passwords stored for you. To test the real nightly run (as
SYSTEM), click **Run now** in the setup window (or run `MivbSync.exe task run`), then read the
newest report.

## The nightly task

The tool manages its own Task Scheduler task, `MivbSync`. Use the setup window's Scheduled task
panel, or these commands from an elevated prompt:

```
MivbSync.exe task install [--time HH:MM]   # create or replace the task (default 02:00)
MivbSync.exe task status                   # installed / enabled / next run / last result
MivbSync.exe task run                      # start a run now
MivbSync.exe task enable | disable         # resume or pause the nightly run
MivbSync.exe task remove                   # delete the task
```

- The task runs daily as SYSTEM, from the install folder, and is stopped after 2 hours. A run
  missed while the machine was off starts when it is back on.
- **After changing a password**: enter it in the setup window and Save while running as
  administrator, or run `set-credential neo|ftp` and then `task install`. SYSTEM has its own
  credential store, and this copies the new password into it.
- **After moving** the exe or `config.toml`: run `task install` again. The task stores absolute
  paths.

## First runs (pilot)

1. **One to two weeks of dry runs.** Set `dry_run = true` at the top of `config.toml`. Each
   night's report then lists what it **would** create and what it flags, without changing Neo.
   Check that against what you know about the estate.
2. **First live run, by hand.** The first live run usually creates many monitor points at once,
   which the safety cap would hold back. Run it once by hand with
   `MivbSync.exe sync --allow-large-change`, then review the report and Neo.
3. **Go live.** Set `dry_run = false`. The nightly task needs no change.
4. **Cleanup (optional).** Once things are stable, set `[cleanup] enabled = true` to archive
   processed exports and delete old ones from the SFTP server.

## Reading the results

Everything is written under `data_dir` (default `C:\ProgramData\MivbSync`):

```
reports\sync-<run id>.txt / .html / .json   one report per run (start here)
logs\sync-<run id>.log                      detailed log per run
audit\audit-YYYY-MM.jsonl                   every monitor point written or deleted in Neo
archive\<controller>\<yyyy-mm-dd>\          copies of processed exports (when cleanup is on)
state.json                                  what the tool remembers between runs
```

About 30 days of history is kept, and older files are removed automatically.

The report shows an overall status (OK, WARN, ERROR or FATAL), then for each controller whether
its export was trusted, and the list of actions: added, flagged, failed, or "would" (dry run or
over a safety limit).

**Exit codes.** These are shown in Task Scheduler's "Last Run Result" (in hex). They are bit
flags, so values add up: 0x9 = fatal + warnings.

| Code | Meaning |
|------|---------|
| 0x0 | OK |
| 0x1 | Completed with warnings (flags, a skipped controller, capped changes) |
| 0x2 | An action failed, or the state could not be saved |
| 0x4 | A notification could not be sent |
| 0x8 | Fatal: the run could not proceed (config, credentials, Neo unreachable, another run active) |

Anything other than 0x0 or 0x1 deserves a look at the newest report. `MivbSync.exe task status`
and the setup window decode the code for you.

## Updates

Installed copies update themselves. At the start of each nightly run, the tool checks this
repository. If there is a newer stable release, it:

1. downloads `MivbSync.exe` next to the current one and checks it against the release's
   `SHA256SUMS.txt`;
2. starts the new exe once as a self-test (`--version`);
3. replaces the current exe, keeping the previous one as `MivbSync.exe.old`;
4. runs that night's sync with the new version. The report says "updated A -> B".

Your `config.toml` is never changed by an update. If anything goes wrong (no connection,
checksum mismatch, failed self-test), the current version stays in place and the sync runs as
normal. The report then shows a warning, or just a note if GitHub could not be reached.

- **Check by hand:** `MivbSync.exe update --check`. Update now: `MivbSync.exe update`, from an
  elevated prompt when the exe is under `C:\Program Files`.
- **Stay on a version, or roll back:** set `pin = "0.4.0"` under `[update]` in `config.toml`.
  The next run installs exactly that version.
- **Turn updates off:** set `enabled = false` under `[update]`.
- **Roll back by hand:** rename `MivbSync.exe.old` back to `MivbSync.exe`.
- **Network:** outbound HTTPS to `api.github.com`, `github.com`, `objects.githubusercontent.com`
  and `release-assets.githubusercontent.com`. If security software blocks executable
  downloads, allow-list those hosts or turn updates off.
- **Setup window:** close it before the nightly run. A window left open while the exe is
  replaced may fail on its next action; just reopen it.

## Configuration reference

`config.toml` is plain text with comments; the downloaded `config.example.toml` documents every
key. After editing, run `MivbSync.exe check-config`.

| Section | Main settings |
|---------|---------------|
| top level | `customer`, `data_dir`, `dry_run` |
| `[neo]` | `server`, `username`, `verify_tls` (false for a self-signed certificate), `timeout_seconds`, `page_size` |
| `[[controllers]]` | one block per primary controller: `name`, `integration`, optional `element_id` |
| `[ftp]` | `host`, `port`, `username`, `folder` (the one folder all controllers export into), optional `host_key_fingerprint`; `protocol` (`sftp` by default; `ftps` / `plain` for older FTP servers) |
| `[export]` | how export files are recognised and read: `filename_regex`, `timezone` (the controllers' local time zone, `"local"` or e.g. `"Europe/London"`), column names under `[export.columns]` |
| `[recording]` | which rows get a monitor point (see below), `reassign_mode` |
| `[safety]` | `max_file_age_hours` (26), `settle_minutes` (10), `max_count_drop_percent` (10), `max_creates_per_run` (100), `max_removals_per_run` (10) |
| `[cleanup]` | `enabled` (false), `keep_latest` (3), `archive`, `archive_retention_days` (30) |
| `[report]` | `subject_prefix`, `retention_days` (30) |
| `[update]` | `enabled` (true), `pin`, `repository`, `timeout_seconds` |
| `[notify]` | optional Asana / Notion notifications, off by default |

**Which rows get a monitor point.** Every row in the export is a candidate, then these filters
apply:

| Rows | Default | Setting |
|------|---------|---------|
| Rows with Device Type `No Device` (unassigned numbers) | skipped | `ignore_device_types` |
| Hot-desk users (Device Type `No Device`, Hot Desking User = Yes) | included | `keep_hot_desk_users` |
| ACD agents | included | `include_acd_agents` |
| Local-only DNs | skipped | `ignore_local_only` |
| Multi-Device User Group member devices (e.g. `3*001`) | included | `ignore_extension_regex = '^.*\*.*$'` skips them |
| Specific numbers | - | `ignore_extensions = ["2000", "2001"]` |

**Extension moved to another integration** (`reassign_mode`):

| Mode | What happens |
|------|--------------|
| `add_only` (default) | Creates the monitor point under the new integration and flags the old one |
| `flag` | Only flags it |
| `move` | Creates it under the new integration, then deletes the old monitor point |

**Safety limits.** An export older than `max_file_age_hours`, still changing within
`settle_minutes`, or with more than `max_count_drop_percent` fewer extensions than last time is
not trusted: that controller is skipped and reported. If a drop is genuine (a site closed),
accept it once with `MivbSync.exe sync --accept-count-drop <controller>`. A run that would create
more than `max_creates_per_run` monitor points creates that many and reports the rest as
"would". Allow them once with `--allow-large-change`.

**Element ids.** Normally every row of a controller's export belongs to that controller. If one
export also lists sets homed on other controllers, set `element_id` on every controller to its
Home Element value. `parse-file` lists the values found in an export. Each row is then assigned
by its Home Element.

## Command reference

| Command | What it does |
|---------|--------------|
| `MivbSync.exe` or `MivbSync.exe setup` | Opens the setup window (`--no-gui` for console prompts) |
| `MivbSync.exe sync` | Runs a sync now (what the nightly task does). Options: `--dry-run`, `--allow-large-change`, `--accept-count-drop CTRL`, `--no-update`, `--no-notify` |
| `MivbSync.exe check-config` | Validates `config.toml`; `--sample-filename NAME` tests an export file name |
| `MivbSync.exe test-connections` | Tests Neo, SFTP and (if enabled) notifications |
| `MivbSync.exe parse-file --controller NAME PATH` | Shows how a local export file is read |
| `MivbSync.exe show-state` | Shows what the tool remembers per controller |
| `MivbSync.exe set-credential neo\|ftp` | Stores a password in Windows Credential Manager for the current user |
| `MivbSync.exe task install\|status\|run\|enable\|disable\|remove` | Manages the nightly task (elevated prompt) |
| `MivbSync.exe update [--check]` | Checks for, or installs, a newer release |
| `MivbSync.exe probe-neo` | Read-only diagnostics of the Neo monitor point API |
| `MivbSync.exe --version` | Prints the version |

Add `--config PATH` before the command to use a config file other than the one beside the exe.

## Troubleshooting

| Symptom | Likely cause | What to do |
|---------|--------------|------------|
| `test-connections`: `[FAIL] login` | Wrong Neo username or password | Check `[neo] username`; enter the password again in the setup window (or `set-credential neo`) |
| `[OK] login` but `[FAIL] grants` | The Neo account may not use the MonitorPoint methods | Use a System Provider superuser; enable the Web Service master switch and the MonitorPoint methods on the System tenant |
| `PERMISSION_FUNCTION_NOT_ALLOWED` | One Web Service method is not granted | Grant that MonitorPoint method on the System tenant's Web Service tab |
| "No '...' credential stored" in the nightly run | The password was never copied to SYSTEM | Save in the setup window (as administrator), or `task install` again |
| "Access is denied" / "Not running as administrator" | Changing the task needs elevation | Right-click > Run as administrator, or accept the UAC prompt |
| Last Run Result 0x80070002 | The exe or config moved | `task install` again from the new location |
| NO_FILES for a controller | No `ascsync` export with that controller name in the folder | Check the export job is called `ascsync` (its File Name under Activity Parameters must be exactly `ascsync`); then check the `[ftp] folder`, and that the controller name matches the file name. `test-connections` lists the exports it found, including any with another File Name |
| STALE | The newest export is too old | Check the MiVB Scheduler job ran; nothing is removed meanwhile |
| UNCHANGED | No new export since the last run | Check the Scheduler job |
| PARTIAL / NOT_SETTLED | The file is incomplete or still being written | Usually fixes itself next run; `parse-file` the copy under `data_dir\runs\<run id>\downloads` |
| COUNT_DROP | Far fewer extensions than last time | Investigate the export; if genuine, `sync --accept-count-drop <controller>` |
| FLAG_UNKNOWN_CONTROLLER | `element_id` is set and a row's Home Element matches no controller | Copy the values `parse-file` lists into the controllers' `element_id`, or remove `element_id` |
| FLAG_MIXED_HOME_ELEMENTS | An export lists sets homed on several controllers | Set `element_id` on every controller, or limit the export to the controller's own sets |
| Hot-desk users or ACD agents missing in Neo | Skipped by the row filters | See "Which rows get a monitor point" |
| "SFTP host key ... seen for the first time" (warning) | First connection to the SFTP server | Compare the logged `SHA256:` fingerprint with the server administrator's |
| "SFTP host key mismatch" | The server's key changed, or this is a different server | Confirm the new key with the server administrator, then delete that host's line in `data_dir\known_hosts` |
| "SFTP authentication failed" | Wrong SFTP username or password | Check `[ftp] username` and enter the password again |
| Report warning "update to X failed" | Download blocked, or the self-test failed | See [Updates](#updates); the current version keeps running |
