<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The script it describes is MIT-licensed.)
-->

# Pre-Migration Script 3 — Identify Broken Inheritance In Your Classes Of Service

**Script:** [`Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh`](Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh) (MIT License)
**Run as:** the `zimbra` user, on a mailbox server of the **source** system, before your dry run.
**Changes anything?** No. It writes a CSV report and a `zmprov -f` reapply batch that you decide what to do with.

## Why this matters

Zimbra policy attributes — `zimbraFeature*`, quotas, protocol toggles — are meant to be managed at the Class of Service level. Setting one on an individual account "breaks" CoS → Account inheritance: the account stops following its CoS for that attribute, silently and forever. Over years of administration these overrides accumulate, and most are either redundant or long-forgotten one-off workarounds.

The migration handles this class in tiers (see [What Migrates — And What Does Not](../1-Evaluate/What-Migrates-And-What-Does-Not.md)):

- Overrides that **tighten security or encode operational policy** — password locks, forwarding-permission policy, per-seat feature grants such as EWS and S/MIME, POP3 toggles, quotas, session lifetimes, 2FA-required/available = TRUE — **migrate automatically**, and every one applied is recorded by the restore in `applied_policy_overrides.txt`. They are *not* in this report.
- Overrides that **loosen policy** (for example a 2FA exemption on an application account), plus provisioning leftovers and UX toggles, are deliberately **not migrated**. **This report lists exactly those**, so nothing surfaces as a surprise during your dry run. Anything an account genuinely depended on is usually better fixed properly — a dedicated CoS, or an app-specific password (which the migration *does* carry) instead of a 2FA exemption — than re-applied.

`zimbraImapEnabled` is excluded entirely: IMAP readiness is [Script 2](Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.md)'s job. User preferences (`zimbraPref*`) are not reported either; users set those through the web client by design, and the migration carries them as user data.

## Running it

```bash
bash Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh --output-dir /var/tmp
```

Options: `--output-dir DIR` (default `/tmp`) and `-h`. It needs `ldapsearch` and `zmlocalconfig`, and it reads LDAP directly — because LDAP knows nothing about inheritance, every attribute it returns is an *explicit* account-level value, which is exactly the override set. Zimbra system accounts and external-virtual share stubs are excluded.

## What it writes

- `mce-cos-overrides_<timestamp>.csv` — one row per override: `account, attribute, value`. **Review this.**
- `mce-reapply-cos-overrides_<timestamp>.zmp` — a `zmprov -f` batch that would re-create every override on the destination. **Applying it re-breaks inheritance deliberately.** Review it line by line; delete what should stay clean; apply only what you conclude is genuinely load-bearing, and only *after* the migration completes:

  ```bash
  zmprov -f /var/tmp/mce-reapply-cos-overrides_<timestamp>.zmp     # on the DESTINATION, after migration, if at all
  ```

The console summary shows the number of overrides found and the number of accounts carrying them.

## Then

Phase Two is complete. Continue to [Phase Three — Get Set](../3-Get-Set/).

---

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
