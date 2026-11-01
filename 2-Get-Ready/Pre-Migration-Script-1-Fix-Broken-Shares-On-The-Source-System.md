<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The script it describes is MIT-licensed.)
-->

# Pre-Migration Script 1 — Fix Broken Shares On The Source System

**Script:** [`Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh`](Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh) (MIT License)
**Run as:** the `zimbra` user, on a mailbox server of the **source** system, before any migration backup.
**Changes anything?** No. It writes CSV reports and removal scripts; nothing is removed unless you run the removal scripts yourself.

## Why this comes first

Over years of use, Zimbra accumulates broken shares: mountpoints to folders that no longer exist, grants to deleted accounts, and stale entries in the `zimbraSharedItem` cache. Left in place, they corrupt mountpoint export and produce wrong IMAPSYNC exclude lists — which can **duplicate shared-folder data** onto the destination, and which show up as noisy log entries and partial failures that nobody can tell apart from real problems. Cleaning them up now moves the noise from mid-cutover (irrecoverable) to a quiet preflight pass where you can review and act with confidence.

## What it detects

1. **Inbound broken shares** (broken mountpoints) — folders in user mailboxes that are mountpoints to shares whose source folder no longer exists or is inaccessible, including "zombie" mountpoints whose owner account was deleted.
2. **Outbound broken shares** (invalid recipients) — shares granted *to* recipients that no longer exist on the system.
3. **Stale ACL entries** — `zimbraSharedItem` cache entries that no longer have a matching grant on the folder's actual ACL (typically a share revoked through the web client without the cache being cleaned up). At restore time these cause `service.PERM_DENIED` when the destination tries to recreate the mountpoint.
4. **Stale acceptance records** — `zimbraSharedItem` entries whose grant is still live but whose grantee no longer has a mountpoint to the folder. Each one would recreate a mountpoint the user deleted (or a duplicate) on the destination.

## Running it

```bash
bash Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh --output-dir /var/tmp
```

Options:

| Option | Default | Purpose |
|---|---|---|
| `--jobs N` | `4` | Parallelism for the account-reachability probe (needs GNU `parallel`; pass `1` for sequential). |
| `--delay SECONDS` | `0.2` | Pause between per-user checks in the inbound scan — raise it on a heavily loaded source. |
| `--timeout SECONDS` | `300` | Per-user `zmsoap` timeout. |
| `--output-dir DIR` | `/var/tmp` | Where the reports, removal scripts, and log are written (timestamp-suffixed). |
| `-h`, `--help` | | Usage. |

Exit codes: `0` no broken shares found · `1` broken shares found (action recommended) · `2` setup or environment error · `3` no reachable users found on this server.

## What it writes

In the output directory, each with a timestamp suffix:

- Four CSV reports: `broken-inbound-shares-*.csv`, `broken-outbound-shares-*.csv`, `stale-acl-shares-*.csv`, `stale-acceptance-shares-*.csv`.
- Four matching removal scripts: `remove-broken-inbound-shares-*.sh`, `remove-broken-outbound-shares-*.sh`, `remove-stale-acl-shares-*.sh`, `remove-stale-acceptance-shares-*.sh`.
- One unified log, `broken-shares-unified-*.log`, and one skipped-accounts file, `broken-shares-skipped-accounts-*.txt` (accounts the reachability probe could not open, with the reason).

## Then

1. **Review the CSVs.** Each row names the account, the folder, and why it was flagged.
2. **Run the removal scripts you agree with**, as the `zimbra` user. They are plain shell scripts — read them first; delete any line you want to keep.
3. Re-run the script. A clean run exits `0`.

When it is clean, continue with [Pre-Migration Script 2 — Ensure All Accounts Are IMAP-Capable](Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.md).

> **Requirements:** `bash`, `zmprov`, `ldapsearch`, `zmmailbox`, `zmsoap`; GNU `parallel` optional. Run inside `screen` or `tmux` — on a large system this takes a while.

---

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
