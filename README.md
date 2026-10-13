# Find-StuckVBMJobs

Find stuck **Veeam Backup for Microsoft 365** (VBM, also called VB365) jobs and
organisation syncs, and raise an RMM alert that names each one and says why.

`Find-StuckVBMJobs.ps1` is a read-only PowerShell script. Run it on the VBM server from
an RMM such as NinjaOne. It checks every backup job, backup copy job and organisation
sync. When something is stuck, the output names the job or organisation, gives the
reason and the times that matter, and the script exits with code `5`.

It changes nothing. It only reads.

## Why

A job that fails is easy to spot. A job that never finishes, never starts, or quietly
stops running is not: it has not failed, so it can go unnoticed for days. The same goes
for an organisation sync that sits in the queue and never starts. This script looks for
exactly those cases.

## Example: stuck jobs found

This is real output from the script, run against test data with made-up names:

```text
VB365 STUCK: 5 problem(s) on VBM-01 (checked 2026-10-13 16:56)
- Job 'Exchange Online Backup' (backup job): has been running for 15.2 hours (started 2026-10-13 01:44). Last backup: 2026-10-12 16:08.
- Job 'SharePoint and OneDrive Backup' (backup job): last run was cut off (Disconnected) at 2026-10-13 07:50. Last backup: 2026-10-12 14:56.
- Job 'Teams Backup' (backup job): missed its schedule: it was due at 2026-10-12 14:56 and did not start; has not run for 50.0 hours (last run 2026-10-11 14:56). Last backup: 2026-10-11 15:26.
- Job 'Archive Mailboxes Backup' (backup job): has been Queued for 26.5 hours (since 2026-10-12 14:26). Last backup: 2026-10-11 15:20.
- Organisation 'contoso.onmicrosoft.com': sync queued since 2026-09-30 13:56 and never started (All, Users, Groups); has not synced for 316.0 hours. Last sync: 2026-09-30 12:56 (Success).
OK: 2 job(s) healthy, 1 organisation(s) syncing normally. INFO: 1 job(s) disabled.
=== RMM SUMMARY ===
Computer=VBM-01; Outcome=STUCK; ExitCode=5; StuckCount=5; Stuck=Exchange Online Backup [running too long]; SharePoint and OneDrive Backup [cut off]; Teams Backup [missed schedule, not run]; Archive Mailboxes Backup [waiting too long]; contoso.onmicrosoft.com sync [never started, not synced]; Detail=; Checked=2026-10-13T16:56:01
```

Exit code: `5`.

* The first line counts the stuck jobs and organisations.
* Then one line per stuck job or organisation: what is wrong, with the times. A job with
  more than one problem gets all its reasons on its one line.
* Then a count of what is healthy, and of disabled jobs.
* The line after `=== RMM SUMMARY ===` is one `Key=Value; ...` line for RMMs that parse
  output. See [The RMM summary line](#the-rmm-summary-line).

With `-ShowAll`, the healthy and disabled items are also listed, before the summary:

```text
OK: 2 job(s) healthy, 1 organisation(s) syncing normally. INFO: 1 job(s) disabled.
  OK    Job 'Public Folders Weekly' (backup job). Last backup: 2026-10-08 17:26.
  OK    Job 'Copy to Object Storage' (backup copy job). Last backup: 2026-10-13 13:08.
  OK    Organisation 'fabrikam.onmicrosoft.com'. Last sync: 2026-10-13 10:56 (Success).
  INFO  Job 'Old Tenant Backup' (backup job): disabled
```

`Public Folders Weekly` runs once a week and last ran 5 days ago. That is normal for its
schedule, so it is not stuck.

## Other results

Nothing stuck (exit `0`):

```text
VB365 OK on VBM-01 (checked 2026-10-13 16:56): 4 job(s) healthy, 2 organisation(s) syncing normally.
=== RMM SUMMARY ===
Computer=VBM-01; Outcome=OK; ExitCode=0; StuckCount=0; Stuck=; Detail=; Checked=2026-10-13T16:56:08
```

The check could not run (exit `1`):

```text
VB365 CHECK FAILED on VBM-01 (checked 2026-10-13 16:56)
- Could not check: could not connect to VB365: The Veeam Backup for Microsoft 365 service is not running.
OK: 0 job(s) healthy, 0 organisation(s) syncing normally.
=== RMM SUMMARY ===
Computer=VBM-01; Outcome=CHECK_FAILED; ExitCode=1; StuckCount=0; Stuck=; Detail=could not connect to VB365: The Veeam Backup for Microsoft 365 service is not running.; Checked=2026-10-13T16:56:10
```

The text after `could not connect to VB365:` is the error from Veeam's own PowerShell
module, so it varies.

VBM is not installed on this server (exit `0`, so a fleet-wide policy does not alert on
servers without VBM):

```text
VB365 is not installed on VBM-01 (checked 2026-10-13 16:56): nothing to check.
=== RMM SUMMARY ===
Computer=VBM-01; Outcome=NOT_APPLICABLE; ExitCode=0; StuckCount=0; Stuck=; Detail=VB365 is not installed; Checked=2026-10-13T16:56:11
```

A machine counts as a VBM server when it has the VBM service (`Veeam.Archiver.Service`)
or its program in Program Files. A machine with only the VBM console or the VBM
PowerShell toolkit also gets this result.

## What counts as stuck

An **enabled** backup job or backup copy job is stuck when:

| Rule | Stuck when | Default |
|---|---|---|
| Running too long | Its latest run is `Running` for more than `MaxRunHours` | 12 hours |
| Waiting too long | Its latest run is `Queued` or `Updating` for more than `MaxWaitHours` | 24 hours |
| Missed its schedule | It is not running, and its next run time passed more than `OverdueMinutes` ago | 60 minutes |
| Not run for too long | It is not running, and its last run is older than `NotRunHours`, or it never ran | 36 hours |
| Last run cut off | Its latest run ended `Disconnected` | - |

* `NotRunHours` is for a daily job. A job that runs on fewer days gets the extra time
  between its runs on top: Workdays +48 hours, Weekends +120 hours, one day a week +144
  hours.
* A job with its schedule turned off runs only by hand. It is never stuck for not
  running, but the other rules still apply.
* An `Immediate` backup copy job follows the schedule of its backup job.
* A new job that waits for its first run is not stuck.
* A disabled job is listed as INFO, never as stuck.

An **organisation sync** is stuck when:

| Rule | Stuck when |
|---|---|
| Never started | It is `Queued`, its scheduled time passed more than `OverdueMinutes` ago, and it never started |
| Running too long | It is `Running` for more than `MaxRunHours` |
| Not synced | Its last sync is older than `NotRunHours`, or it never synced. Not while a sync runs within its limit |

* A part (Users, Groups, Sites and so on) that waits its turn in a sync that is running
  within its limit is not stuck.
* Only Microsoft 365 organisations that use modern app-only authentication have this sync.
  On-premises organisations, and Microsoft 365 organisations still connected with basic
  authentication or modern authentication with legacy protocols, are not checked.

Times are compared in the server's local time.

## Requirements

* Windows, on the VBM server itself (the server with the Veeam Backup for Microsoft 365
  service).
* Veeam Backup for Microsoft 365 version 8 or later. Its PowerShell module needs
  PowerShell 7.4.2 or later.
* PowerShell 7 (`pwsh.exe`) installed. You can start the script from Windows PowerShell
  5.1: it restarts itself under PowerShell 7 and passes the exit code back. On a server
  without VBM, PowerShell 7 is not needed.
* Run it as SYSTEM or as a local administrator.

## Quick start

On the VBM server, in an elevated Windows PowerShell window:

```powershell
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12; New-Item -ItemType Directory -Force C:\Temp | Out-Null; Invoke-WebRequest -UseBasicParsing https://raw.githubusercontent.com/acgdickie/Find-StuckVBMJobs/main/Find-StuckVBMJobs.ps1 -OutFile C:\Temp\Find-StuckVBMJobs.ps1
```

```powershell
powershell -ExecutionPolicy Bypass -File C:\Temp\Find-StuckVBMJobs.ps1; "Exit code: $LASTEXITCODE"
```

Add `-ShowAll` to also list the healthy and disabled items.

## Exit codes

| Exit | Meaning |
|---|---|
| `0` | Nothing is stuck. Or VBM is not installed on this server: nothing to check |
| `5` | Something is stuck. The output names each job or organisation and why |
| `1` | The check could not run. For example: VBM is installed but PowerShell 7 is not, a setting is not valid, the script could not connect to VBM, or the RMM ran the script text instead of a `.ps1` file. The output says why |

When something is stuck and part of the check also failed, the exit code is `5`. The
output lists both, but the summary line names only the stuck items.

## Settings

| Parameter | Default | Meaning |
|---|---|---|
| `-MaxRunHours` | `12` | A job run or a sync is stuck after `Running` this many hours |
| `-MaxWaitHours` | `24` | A job run is stuck after `Queued` or `Updating` this many hours |
| `-NotRunHours` | `36` | A daily job, or a sync, is stuck when its last run is older than this |
| `-OverdueMinutes` | `60` | Grace time after a job's next run time, or a sync's scheduled time |
| `-NinjaField` | empty | NinjaOne only: the device custom field to write the result to |
| `-ShowAll` | off | Also list the healthy and disabled items |

A setting not given on the command line comes from an environment variable with the same
name. This is how NinjaOne Script Variables arrive. A command-line value wins. A value
that is not valid (for example `abc`, or `0`) is a check failure, exit `1`.

```powershell
powershell -ExecutionPolicy Bypass -File C:\Temp\Find-StuckVBMJobs.ps1 -MaxRunHours 8 -NotRunHours 48
```

## Run it from any RMM

* Run it on the VBM server, as SYSTEM, with Windows PowerShell (64-bit).
* Upload `Find-StuckVBMJobs.ps1` only. It needs no other file.
* Run it from a saved `.ps1` file, as NinjaOne and most RMMs do. If your RMM pastes the
  script text into Windows PowerShell instead, the script cannot restart under PowerShell
  7, and the check fails with exit `1` ("the script is not running from a file"). In that
  case, run it with PowerShell 7 (`pwsh.exe`) directly.
* Run it on a schedule, for example every 4 hours.
* Alert on exit `5` (stuck). Alert on exit `1` too (the check could not run), perhaps at a
  lower severity.
* Put the script output in the alert. The stuck lines are at the top, so the alert says
  which job to look at.

## Run it from NinjaOne

1. **Add the script.** In the NinjaOne automation library, add a new script: language
   PowerShell, operating system Windows, architecture 64-bit (or All), run as System.
   Paste in the whole of `Find-StuckVBMJobs.ps1`.
2. **Optional: change the settings.** Add Script Variables named `maxRunHours`,
   `maxWaitHours`, `notRunHours`, `overdueMinutes` (integer), `ninjaField` (string) and
   `showAll` (checkbox). Leave one out to use its default.
3. **Optional: show the result on the device.** Add a **device** custom field of type
   **Multi-line**, with the **Automations** permission set to allow writing (Read/Write or
   Write Only). For example: label `VBM Stuck Status`, name `vbmStuckStatus`. Put the
   field's **name** (not its label) in the `ninjaField` Script Variable. Each run then
   writes the alert text to that field. A failed field write only prints a warning: it
   never changes the exit code.
4. **Alert.** In a policy, add a **Script Result** condition that runs the script on a
   schedule, for example every 4 hours.
   * Set **Timeout** to a few minutes, for example 5. NinjaOne stops a script that runs
     longer.
   * Set the result code to `5`.
   * NinjaOne also checks **With Output**, and both must match. Leave its text empty and
     choose *Does Not Contain*, so only the result code counts.
   * Tick **Include script output in final alert**, so the alert names the stuck jobs.
   * If you also want to know when the check could not run, add a second condition, set up
     the same way, for result code `1`.

NinjaOne shows every non-zero exit code as a failed script. For this script, a "failed"
run with exit `5` means it worked and found something stuck.

### What the custom field shows

A **Multi-line** field gets the alert text, without the summary line:

```text
VB365 STUCK: 5 problem(s) on VBM-01 (checked 2026-10-13 16:56)
- Job 'Exchange Online Backup' (backup job): has been running for 15.2 hours (started 2026-10-13 01:44). Last backup: 2026-10-12 16:08.
- Job 'SharePoint and OneDrive Backup' (backup job): last run was cut off (Disconnected) at 2026-10-13 07:50. Last backup: 2026-10-12 14:56.
- Job 'Teams Backup' (backup job): missed its schedule: it was due at 2026-10-12 14:56 and did not start; has not run for 50.0 hours (last run 2026-10-11 14:56). Last backup: 2026-10-11 15:26.
- Job 'Archive Mailboxes Backup' (backup job): has been Queued for 26.5 hours (since 2026-10-12 14:26). Last backup: 2026-10-11 15:20.
- Organisation 'contoso.onmicrosoft.com': sync queued since 2026-09-30 13:56 and never started (All, Users, Groups); has not synced for 316.0 hours. Last sync: 2026-09-30 12:56 (Success).
OK: 2 job(s) healthy, 1 organisation(s) syncing normally. INFO: 1 job(s) disabled.
```

Use a Multi-line field. A **Text** field holds only 200 characters. When NinjaOne refuses
the full text, the script writes this short version instead, cut at 200 characters:

```text
STUCK 5: Exchange Online Backup [running too long]; SharePoint and OneDrive Backup [cut off]; Teams Backup [missed schedule, not run]; Archive Mailboxes Backup [waiting too long]; contoso.onmicroso...
```

When nothing is stuck, the field shows the one `VB365 OK ...` line. It is short enough
for a Text field too.

## The RMM summary line

The line right after `=== RMM SUMMARY ===` is always one line of `Key=Value` pairs. If a
NinjaOne field write fails, a `WARNING:` line can come after it, so find the summary by
the marker, not by taking the last line.

| Key | Value |
|---|---|
| `Computer` | The server name |
| `Outcome` | `OK`, `STUCK`, `CHECK_FAILED` or `NOT_APPLICABLE` (VBM not installed) |
| `ExitCode` | `0`, `5` or `1`, the same as the exit code |
| `StuckCount` | How many jobs and organisations are stuck |
| `Stuck` | Each stuck item as `name [tags]`, separated by `; `. An organisation sync ends in `sync` |
| `Detail` | Why the check could not run (`CHECK_FAILED`), or `VB365 is not installed` (`NOT_APPLICABLE`). Empty otherwise |
| `Checked` | When the check ran, in local time (`yyyy-MM-ddTHH:mm:ss`) |

The tags are: `running too long`, `waiting too long`, `missed schedule`, `not run`,
`never run`, `cut off` (jobs), and `never started`, `sync running too long`, `not
synced`, `never synced` (organisation syncs).

`Stuck` and `Detail` can contain `; ` themselves, so do not split the whole line on `; `.
The keys always come in the order above. Read one key with a pattern, for example
`Outcome=(\w+)`, `ExitCode=(\d+)` or `StuckCount=(\d+)`, and take `Stuck` as the text
between `Stuck=` and `; Detail=`.

## Safe to run

* **Read-only.** It only connects to VBM and reads jobs, their latest run, organisations
  and sync states. It never starts, stops or changes anything.
* **Fails closed.** If it cannot check, it says so and exits `1`. It never reports OK for
  something it could not read.
* **One file, no install.** No modules to install, no scheduled tasks, no files left
  behind.

## Limits

* **VBM only.** It does not check Veeam Backup & Replication.
* **Failed runs are not "stuck".** A job whose last run ended `Failed` or `Warning` did
  run. Use VBM's own notifications, or your RMM's Veeam monitoring, for failed runs.
* **Only the latest run of each job is checked.**
* **Long first runs.** The first full backup of a large organisation, or the first copy to
  object storage, can run for days. Expect a `running too long` alert during it, or raise
  `-MaxRunHours` for that server until it finishes.
* **Backup windows.** A job that runs periodically, with a backup window that blocks whole
  days, can go longer than `NotRunHours` between runs. Raise `-NotRunHours` for that
  server.
* **Tested on VBM 8.** Older versions are not tested.

## License

[MIT](LICENSE).

Veeam and Microsoft are trademarks of their owners. This project is not affiliated with
or endorsed by Veeam or Microsoft.
