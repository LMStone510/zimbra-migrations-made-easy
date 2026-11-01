<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The script it describes is MIT-licensed.)
-->

# Pre-Migration Script 2 — Ensure All Accounts Are IMAP-Capable

**Script:** [`Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh`](Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh) (MIT License)
**Run as:** the `zimbra` user, on a mailbox server of the **source** system, before any migration backup.
**Changes anything?** No. It writes a `zmprov -f` provisioning file that you review and apply yourself.

## Why this matters

IMAPSYNC copies mail by logging into each mailbox over IMAP. Any account with `zimbraImapEnabled=FALSE` cannot be authenticated, and **its email folder structure and mail blobs are silently skipped and not migrated**. Some Classes of Service ship with IMAP off, and operators often disable it over the years on admin accounts, service accounts, and anywhere a hardening policy was applied.

## What it does

It queries Zimbra's local LDAP directly (one query for all Classes of Service, one for all accounts — fast, no per-account `zmprov` calls) and finds:

- every **Class of Service** with `zimbraImapEnabled=FALSE`, and
- every **account** with an *explicit* account-level `zimbraImapEnabled=FALSE` override.

It then writes a single bulk-provisioning file containing the `zmprov` commands to turn IMAP on. The script is careful about CoS inheritance: accounts whose IMAP is off only because their CoS says so are fixed by the CoS-level line, and no per-account override is emitted for them — so you do not end up with spurious account-level settings that shadow future CoS changes.

> **External-account Classes of Service are correctly excluded.** Zimbra's `defaultExternal` CoS (and any custom external CoS you add to `EXTERNAL_COS_SKIP` near the top of the script) is applied only to *external virtual share-target* accounts — stubs created when a user shares a folder to an outside address. Those should **not** have IMAP enabled, so the script reports them as excluded rather than flagging them.

## Running it

```bash
bash Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh --output-dir /tmp
```

Options:

| Option | Default | Purpose |
|---|---|---|
| `-o <path>` | | Full path of the output file (overrides `--output-dir`). |
| `--output-dir DIR` | `/tmp` | Directory for the auto-named output file, `mce-imap-preflight_<timestamp>.zmp`. |
| `-h`, `--help` | | Usage. |

Exit codes: `0` nothing to fix (IMAP already enabled everywhere it should be) · `1` fixes were generated (review and apply) · `2` setup or environment error.

## Then

Review the generated file, then apply it on the source:

```bash
zmprov -f /tmp/mce-imap-preflight_<timestamp>.zmp
```

Re-run the script; a clean run exits `0`. The migration software will ask you to confirm this has been done at the start of every `pre` and `full` backup.

Optionally, run it again on the **destination** after the restore so the migrated accounts have IMAP available to end users.

Next: [Pre-Migration Script 3 — Identify Broken Inheritance In Your Classes Of Service](Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.md).

> **Requirements:** `bash`, `zmprov`, `ldapsearch`.

---

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
