<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The scripts in this directory are
  separately MIT-licensed; this notice does not apply to them.)
-->

# Phase Two — Get Ready

Three free, open-source (MIT) scripts that find — and help you fix — the things that break Zimbra migrations. Run them on your **source** system, as the `zimbra` user, **in this order**, before any backup is taken:

| # | Script | What it finds | Guide |
|---|---|---|---|
| 1 | [`Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh`](Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh) | Broken mountpoints, grants to deleted accounts, stale share-cache entries — the things that corrupt share export and cause duplicated mail on the destination. | [Usage guide](Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.md) |
| 2 | [`Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh`](Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh) | Every Class of Service and account with IMAP disabled — accounts whose mail would otherwise be silently skipped. | [Usage guide](Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.md) |
| 3 | [`Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh`](Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh) | Every account-level override of a Class-of-Service policy attribute that the migration will deliberately not carry — your review list before the dry run. | [Usage guide](Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.md) |

**All three are read-only.** Scripts 1 and 2 generate removal scripts or a provisioning file that *you* review and apply; Script 3 only reports. None of them changes your system on its own.

The migration software will ask you, at the start of every backup, to confirm that Scripts 1 and 2 have been run and their fixes applied. Do not answer "yes" until they have.

## Requirements

- `bash` and `zmprov` (all three scripts).
- Script 1 additionally needs `ldapsearch`, `zmmailbox`, and `zmsoap`; GNU `parallel` is optional (it falls back to sequential probing).
- Script 2 additionally needs `ldapsearch`.
- Script 3 needs `ldapsearch` and `zmlocalconfig`.
- Run inside a `screen` or `tmux` session — Script 1 can take a while on a large system.

## Not migrating?

Script 1 is worth running periodically on any long-lived Zimbra system: broken shares accumulate as users come and go, and they produce noisy logs and confusing behaviour long before anyone plans a migration.

---

When all three have been run and their fixes applied, continue to [Phase Three — Get Set](../3-Get-Set/).

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
