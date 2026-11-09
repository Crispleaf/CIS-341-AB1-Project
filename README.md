# CIS-341-AB1-Project
Linux System Health Monitor and Security Auditing Daemon

SysAudit is a small program that watches the health and security of a Linux computer. Every 5 minutes it saves a report (a `.audit` file) in `/course_project/audit`. At 00:00 every 3 days, a systemd timer packs the reports into one `.tar.gz` archive and deletes the raw `.audit` files, but only after it has checked that the archive is good. It also deletes archives older than 14 days and writes an `ARCHIVE SUMMARY` line to its log. The program keeps its own log in `/var/log/sysaudit_daemon.log` with the levels `[INFO]`, `[WARN]` and `[ERROR]`. When the workspace reaches `max_storage_mb`, it writes a `[WARNING]` line. Only the admin account `auditmanager`, or a user it has delegated, may run it. Every kind of failure has its own exit code (11 codes, 0 to 10).

## Team and files

| Member | Files |
|---|---|
| Member 1 | `main.py`, `modules/config.py`, `sysaudit.cfg` |
| Member 2- Javvad Mohammed| `modules/audit.py`, `systemd/`, ReadMe, PPT |
| Member 3 | `modules/security.py` |
| Member 4 | `modules/monitoring.py` |
| Member 5 | `tests/` |
| ALL | PPT, ReadMe |

## Architecture
What/who starts the program, the modules that get used. The files that get read or are outputted. The basic architecture or "flowchart" of the system:
<img width="1334" height="646" alt="image" src="https://github.com/user-attachments/assets/37f68f27-c571-4f6d-b963-3dcb83809349" />
Stages of one audit run has these 7 basic step, :
<img width="1388" height="283" alt="image" src="https://github.com/user-attachments/assets/a6b7e852-4233-4e31-9c2e-51211ba86da6" />


## Project structure

```
<img width="508" height="384" alt="image" src="https://github.com/user-attachments/assets/fd1966a6-4179-435f-adaf-24db01a09ffa" />

CIS341-System-Audit-Project/
├── main.py                                  main program you run
├── sysaudit.cfg                             all the settings
├── modules/
│   ├── __init__.py                          marks modules/ as a Python package
│   ├── config.py                            reads and checks sysaudit.cfg
│   ├── audit.py                             scan report, archive, prune, summary
│   ├── security.py                          who may run it, delegation
│   └── monitoring.py                        log file, workspace check, storage ceiling
├── systemd/
│   ├── sysaudit.service                     starts the daemon at boot
│   ├── sysaudit-cleanup.service             one cleanup run
│   └── sysaudit-cleanup.timer               runs the cleanup at 00:00 every 3 days
├── tests/
│   └── test_sysaudit.py          
├── docs/
│   └── test_results_sample.txt              results of our own test run 
├── README.md                     (shared)   this file
└── .gitignore                    (shared)   files git should skip
```
# Basic setup:
## 1. Prepare the VM
Use the latest Ubuntu in VirtualBox.
1. Take a snapshot of the VM before (Machine > Take Snapshot). This can be used to roll back incase of crashes
2. Update the package list and install the tools. `acl` gives `setfacl` and `getfacl`, `iproute2` gives `ss`, and `unzip` opens the project zip file.

```
sudo apt update
sudo apt install -y python3 acl iproute2 unzip
```

## 2. Install

Copy `CIS341-System-Audit-Project.zip` into your home folder on the VM , then unzip it and go into the folder:

```
cd ~
unzip CIS341-System-Audit-Project.zip
cd CIS341-System-Audit-Project
```

Create the account that is allowed to run the program. Adding it to the `adm` group lets it read the system logs:

```
sudo adduser auditmanager
sudo usermod -aG adm auditmanager
```

Copy the files and make the folders. The `systemd/` folder is copied too, because the tests read the unit files from there when they are not installed in `/etc/systemd/system` yet.

program code, only auditmanager can run it
```
sudo mkdir -p /opt/sysaudit
sudo cp -r main.py modules tests systemd /opt/sysaudit/
sudo chmod 755 /opt/sysaudit/main.py /opt/sysaudit/tests/test_sysaudit.py
```
copy settings
```
sudo mkdir -p /etc/sysaudit
sudo cp sysaudit.cfg /etc/sysaudit/
sudo chmod 644 /etc/sysaudit/sysaudit.cfg
```
delegate list folder
```
sudo mkdir -p /var/lib/sysaudit
sudo chown auditmanager:auditmanager /var/lib/sysaudit
sudo chmod 755 /var/lib/sysaudit
```
workspace and log
```
sudo mkdir -p /course_project/audit
sudo chown auditmanager:auditmanager /course_project/audit
sudo chmod 750 /course_project/audit
sudo touch /var/log/sysaudit_daemon.log
sudo chown auditmanager:auditmanager /var/log/sysaudit_daemon.log
sudo chmod 640 /var/log/sysaudit_daemon.log
```

### Check that it installed:

```
/opt/sysaudit/main.py --help | head -3
```
If the help menu/options/information shows, the program is installed.

## 3. Configure

All settings are in `/etc/sysaudit/sysaudit.cfg` (INI layout: `[SECTION]` headers, then `key=value` lines, no spaces around `=`, `#` starts a comment).

| Section | Key | Default | Allowed | Meaning |
|---|---|---|---|---|
| `[PATHS]` | `workspace` | `/course_project/audit` | absolute path | where `*.audit` reports go |
| `[PATHS]` | `archive_subdir` | `archives` | name | archive folder inside the workspace |
| `[PATHS]` | `log_file` | `/var/log/sysaudit_daemon.log` | absolute path | the daemon's own log |
| `[PATHS]` | `auth_log` | `/var/log/auth.log` | path | used to count failed logins |
| `[STORAGE]` | `max_storage_mb` | `100` | 1-100000 | storage ceiling (MB) |
| `[STORAGE]` | `stop_on_limit` | `no` | `yes` or `no` | `yes` = exit 5 at the ceiling of storage limit, `no` = warn only |
| `[STORAGE]` | `archive_retention_days` | `14` | 1-3650 | delete archives older than this |
| `[STORAGE]` | `disk_alert_percent` | `90` | 1-100 | warn if `/` is fuller than this |
| `[SCHEDULE]` | `scan_interval_seconds` | `300` | 10-86400 | amount of seconds between daemon scans |
| `[NAMES]` | `report_prefix` | `health` | name | health report file name prefix |
| `[NAMES]` | `archive_prefix` | `audit` | name | archive file name prefix |
| `[SECURITY]` | `delegate_permissions` | `rwX` | `rwX` or `rX` | ACL given by `--delegate` |

There are no defaults, keys must be in the file. An unknown, missing or repeated key stops the program with exit 6.

#### Precedence: command line beats the file
A command-line option/flag beats the value in the file for that one run. The file is remains unchanged.

| Option | What it overrides |
|---|---|
| `-c`, `--config FILE` | the config file to read (default `/etc/sysaudit/sysaudit.cfg`) |
| `-w`, `--workspace DIR` | `[PATHS] workspace` |
| `-l`, `--log-file FILE` | `[PATHS] log_file` |
| `-m`, `--max-storage-mb N` | `[STORAGE] max_storage_mb` (1-100000) |
| `-r`, `--retention-days N` | `[STORAGE] archive_retention_days` (1-3650) |
| `-i`, `--interval-seconds N` | `[SCHEDULE] scan_interval_seconds` (10-86400) |

Example: the config says 100 MB, but this one run uses 1 MB.
```
sudo -u auditmanager /opt/sysaudit/main.py --scan -m 1
```

### How to Run: Example
Always start the program as `auditmanager` (or a delegate). Pick exactly one mode per run.
## Get help: --help
Works for every user (exit 0).
```
/opt/sysaudit/main.py --help
```
Output looks like:
```
==========================================================================
 SYSAUDIT - System Health Monitor & Security Auditing Daemon  (CIS 341)
==========================================================================

USAGE
  main.py                       show this help (no arguments)
  main.py -h | --help           show this help
  main.py MODE [OPTIONS]        run one mode
...
```

#### To scan: --scan
Run a scan.
```
sudo -u auditmanager /opt/sysaudit/main.py --scan
```
Saves one health and security report, then checks the storage ceiling.
```
2026-11-08 10:00:00 [INFO] Scan started by auditmanager
2026-11-08 10:00:00 [INFO] Scan report saved: /course_project/audit/health_20261108_100000.audit (2154 bytes)
2026-11-08 10:00:00 [INFO] Storage check: workspace uses 0 MB (312 KB) of 100 MB
```

#### Perform a cleanup:  --cleanup
Packs all `.audit` files into one `.tar.gz`, deletes the raw files after the archive is checked, deletes old archives, and logs the summary.
```
sudo -u auditmanager /opt/sysaudit/main.py --cleanup
```
Output example:
```
2026-11-09 01:38:56 [INFO] Cleanup started by auditmanager in /course_project/audit
2026-11-09 01:38:56 [INFO] Archive created: audit_20261109_013856.tar.gz (3 files, 728 bytes)
2026-11-09 01:38:56 [INFO] Deleted 3 raw .audit files after successful compression
2026-11-09 01:38:56 [INFO] Pruned old archive: audit_old20.tar.gz (older than 14 days)
2026-11-09 01:38:56 [INFO] ARCHIVE SUMMARY: total_archives=2 largest=audit_20261109_013856.tar.gz size_bytes=728 modified=2026-11-09 01:38:56
...
```
#### To get Status --status
Shows the workspace size, the number of archives and the delegates.
```
sudo -u auditmanager /opt/sysaudit/main.py --status
```
Output example:
```
SysAudit status
  Workspace      : /course_project/audit
  ...
  Delegated users: student1
```
### Auto run at interval: --daemon
Runs the scripts/program forever and performs scan every specified seconds. Scan, sleep `scan_interval_seconds`, repeat. 

```
sudo -u auditmanager /opt/sysaudit/main.py --daemon -i 60
```
It also re-reads the config file when the file changes. Stop it with Ctrl+C, or with `systemctl stop` when it runs as a service.
```
2026-11-08 10:00:00 [INFO] Daemon started (PID 812), scan every 60 seconds, config /etc/sysaudit/sysaudit.cfg
2026-11-08 10:00:00 [INFO] Scan loop #1 started
2026-11-08 10:00:00 [INFO] Scan started by auditmanager
2026-11-08 10:00:00 [INFO] Scan report saved: /course_project/audit/health_20261108_100000.audit (2154 bytes)
2026-11-08 10:00:00 [INFO] Storage check: workspace uses 0 MB (312 KB) of 100 MB
2026-11-08 10:00:00 [INFO] Sleep cycle: next scan in 60 seconds
...
```
## 4. Run it as a service
Copy the three unit files (from /systemd/) and tell systemd to read them:

```
sudo cp systemd/sysaudit.service systemd/sysaudit-cleanup.service systemd/sysaudit-cleanup.timer /etc/systemd/system/
sudo systemctl daemon-reload
```
Start the daemon now and at every boot, and turn on the cleanup timer:
```
sudo systemctl enable --now sysaudit.service
sudo systemctl enable --now sysaudit-cleanup.timer
```
Check that both are running:
```
systemctl status sysaudit.service
systemctl list-timers sysaudit-cleanup.timer
```
See the next 5 run times of the timer (day 1, 4, 7, ... of each month at midnight):
```
systemd-analyze calendar --iterations=5 "*-*-1/3 00:00:00"
```
Watch the log and the system journal:
```
sudo tail -f /var/log/sysaudit_daemon.log
journalctl -u sysaudit.service --no-pager | tail
```
Test the cleanup now, without waiting for midnight:
```
sudo systemctl start sysaudit-cleanup.service
```
### Cron alternative
Cron can run the cleanup instead of the timer. Open the crontab of `auditmanager`:
```
sudo crontab -u auditmanager -e
```
Add this line, save and exit:
```
0 0 */3 * * /usr/bin/python3 /opt/sysaudit/main.py --cleanup
```
Note: use either the timer OR cron, not both. If you use both, the cleanup could run twice.
## 5. Delegate access to another user
Create a test user. It cannot run the program yet (exit 2):
```
sudo adduser student1
sudo -u student1 /opt/sysaudit/main.py --scan
```
Let `auditmanager` delegate `student1`, then look at the delegate list and the ACLs (extra file permissions):
```
sudo -u auditmanager /opt/sysaudit/main.py --delegate student1
```
view the delegate list and the ACLs
```
cat /var/lib/sysaudit/delegates.txt
getfacl /course_project/audit
```
Now `student1` can scan and see the status:
```
sudo -u student1 /opt/sysaudit/main.py --scan
sudo -u student1 /opt/sysaudit/main.py --status
```
Take the access back:
```
sudo -u auditmanager /opt/sysaudit/main.py --revoke student1
```
A delegate can run the program and use the workspace and the log. A delegate cannot add or remove delegates. Root, system accounts and unknown users are refused with exit 9.
If you set `delegate_permissions=rX` in the config before you delegate, delegates are read-only. Their scans then stop with exit 4.
## 6. Run the tests

```
cd ~
sudo python3 /opt/sysaudit/tests/test_sysaudit.py
tail -5 test_results.txt
```

What to know:

- It needs `sudo` because it creates two temporary users (`sysaudit_intruder`, `sysaudit_helper`) and switches accounts.
- It works in its own sandbox `/tmp/sysaudit_tests` with its own config and log, so the real workspace and log are not touched.
- It saves and restores the real delegate list.
- It removes the test users at the end.
- The results are saved to `test_results.txt` in the folder you ran it from.

Our own run is saved in `docs/test_results_sample.txt`. Expected last line:

```
RESULT: 92 passed, 0 failed, 92 total
```

Which tests cover which requirement:

| # | Requirement | Tests |
|---|---|---|
| 1 | USAGE engine | T01-T05, T72, T73 |
| 2 | systemd service at boot | T87, T88 |
| 3 | Cleanup every 3 days (archive, delete raw) | T47-T51, T58, T59, T63, T89-T91 |
| 4 | External config sysaudit.cfg | T11-T23, T74, T75 |
| 5 | Own log with [INFO]/[WARN]/[ERROR] | T37, T38, T77, T80-T86 |
| 6 | Prune archives older than 2 weeks | T52-T54, T76 |
| 7 | Archive summary line | T55-T57 |
| 8 | Storage ceiling [WARNING] | T39-T43 |
| 9 | CLI beats config | T24-T29, T39, T44-T46, T76 |
| 10 | Errors to STDERR, unique exit codes, no crash | T09, T30-T34, T58, T71, T77, T92 |
| 11 | Only auditmanager may run it | T06-T10, T69 |
| 12 | Delegation | T60-T70, T78, T79 |
| 13 | Automatic test suite | all 92 tests |
| 14 | Presentation | slides (not testable by code) |

The 14 test groups:

1. USAGE engine (T01-T05)
2. Sandbox hacks (T06-T10)
3. Missing and malformed config (T11-T18)
4. Boundary values and bad command lines (T19-T29)
5. Bad paths (T30-T34)
6. Scan, log format, ceiling (T35-T43)
7. Precedence (T44-T46)
8. Cleanup (T47-T59)
9. Delegation (T60-T70)
10. Broken installation (T71)
11. More requirement checks: help, config names, precedence, error log, ACLs (T72-T79)
12. Daemon mode and log events (T80-T86)
13. systemd service and timer files (T87-T91)
14. Crash check, no Python traceback in any run (T92)

# Live demo example:

1. Help works for anyone (exit 0).

   ```
   /opt/sysaudit/main.py --help | head -20
   ```

2. Your own user is refused (exit 2).

   ```
   /opt/sysaudit/main.py --scan
   echo $?
   ```

3. Report saved, and the storage line.

   ```
   sudo -u auditmanager /opt/sysaudit/main.py --scan
   ```

4. Storage ceiling: show the `[WARNING]` line. The CLI option `-m` beats the config.

   ```
   sudo -u auditmanager dd if=/dev/zero of=/course_project/audit/filler.bin bs=1M count=2
   sudo -u auditmanager /opt/sysaudit/main.py --scan -m 1
   sudo rm /course_project/audit/filler.bin
   ```

5. Cleanup: archive, delete raw files, prune, ARCHIVE SUMMARY.

   ```
   sudo -u auditmanager /opt/sysaudit/main.py --cleanup
   ```

6. Delegation: give access, show that the delegate can scan, take it back.

   ```
   sudo -u auditmanager /opt/sysaudit/main.py --delegate student1
   sudo -u student1 /opt/sysaudit/main.py --scan
   sudo -u auditmanager /opt/sysaudit/main.py --revoke student1
   ```

7. The service is running, and the timer shows the next run.

   ```
   systemctl status sysaudit.service
   systemctl list-timers sysaudit-cleanup.timer
   ```

8. Show the daemon's own log.

   ```
   sudo tail -n 15 /var/log/sysaudit_daemon.log
   ```

9. Show the saved test result instead of running the full suite live.

   ```
   tail -5 ~/test_results.txt
   ```


## Exit codes

| Code | Name | Meaning |
|---|---|---|
| 0 | SUCCESS | the task finished without errors |
| 1 | GENERIC_ERROR | a module is missing, or an unexpected error happened |
| 2 | IDENTITY_VIOLATION | the user is not auditmanager or a delegate |
| 3 | CONFIGURATION_MISSING | sysaudit.cfg was not found or cannot be read |
| 4 | INVALID_WORKSPACE | workspace is missing, not a directory, or not writable |
| 5 | DISK_CEILING_REACHED | storage ceiling reached and stop_on_limit=yes |
| 6 | CONFIGURATION_INVALID | a section, key or value in sysaudit.cfg is wrong |
| 7 | INVALID_ARGUMENT | unknown option, missing value or bad value |
| 8 | FILE_OPERATION_FAILED | a report or archive could not be written |
| 9 | DELEGATION_FAILED | --delegate/--revoke failed (bad user, not owner, ACL error) |
| 10 | LOG_UNAVAILABLE | the log file cannot be created or written |

Codes 0 to 5 follow the project guide's suggested table; 6 to 10 were added so every kind of failure has its own code. Every error prints one line `sysaudit: ERROR [exit N NAME]: reason` to STDERR and, once the log is open, to the log.

## Troubleshooting

| Problem | Fix |
|---|---|
| Exit 2 IDENTITY_VIOLATION | Run with `sudo -u auditmanager`, or ask auditmanager to delegate you. |
| Exit 3 | Check the path given with `-c`, or that `/etc/sysaudit/sysaudit.cfg` exists and is readable. |
| Exit 4 | The workspace is missing or has the wrong owner: `sudo chown auditmanager:auditmanager /course_project/audit` |
| Exit 6 | The message names the line or key that is wrong. |
| Exit 10 | Fix the log owner: `sudo chown auditmanager:auditmanager /var/log/sysaudit_daemon.log` |
| "setfacl not found" | acl might not be installed, use to install `sudo apt install acl` |
| "(ss not available)" in a report | `sudo apt install iproute2` |
| A delegate's scan shows "[WARN] Cannot read /var/log/auth.log" | This is normal. The delegate is not in the `adm` group, so the program counts failed logins from `journalctl` instead. |
| The service keeps restarting | `journalctl -u sysaudit.service` and the log show the reason. |

## Uninstall

```
sudo systemctl disable --now sysaudit.service sysaudit-cleanup.timer
sudo rm /etc/systemd/system/sysaudit.service /etc/systemd/system/sysaudit-cleanup.service /etc/systemd/system/sysaudit-cleanup.timer
sudo systemctl daemon-reload
sudo rm -r /opt/sysaudit /etc/sysaudit /var/lib/sysaudit
```

The reports and the log are kept unless you also delete `/course_project/audit` and `/var/log/sysaudit_daemon.log`.
