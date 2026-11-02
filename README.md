<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The four preparation scripts in
  this repository are separately MIT-licensed; this notice does not apply to them.)
-->

# Zimbra Migrations Made Easy

**Any version 8.8.15 or higher. Any OS. Any Zimbra edition. All easily migrated — with the source left intact.**

When a Rolling Upgrade or a system-to-system rsync just won't do, the Mission Critical Email migration software moves a complete Zimbra system onto fresh infrastructure — recreating every account, user setting, and mailbox on the destination in staged passes that keep users live on the source until a short, final cutover — all while the original system stays fully intact as your fallback.

This repository is the companion to <https://www.missioncriticalemail.com/zimbra-migrations-made-easy/>. It walks you through a migration in **four phases**. Each phase has its own directory; work through them in order.

| Phase | Directory | What you do there |
|---|---|---|
| **1. Evaluate** | [`1-Evaluate/`](1-Evaluate/) | Decide whether the software fits your migration, and satisfy your security and audit teams that it is well-tested and data-safe. |
| **2. Get Ready** | [`2-Get-Ready/`](2-Get-Ready/) | Run the three free, open-source pre-migration scripts on your source system to find and fix the things that break migrations. |
| **3. Get Set** | [`3-Get-Set/`](3-Get-Set/) | Prepare the IMAPSYNC server, determine how many seats to license, and request your contract and licensing agreement. |
| **4. Go** | [`4-Go---The-Operator-Guide/`](4-Go---The-Operator-Guide/) | The Operator Guide: the step-by-step procedure for performing the migration with your licensed build. |

---

## Phase One — Evaluate

Is this the right tool for your migration, and can you trust it with your data?

- [**Read me first**](1-Evaluate/README.md) — when to reach for this software, how the two-path architecture works, and how we back up the claim that it is well-tested and data-safe.
- [**What migrates — and what doesn't**](1-Evaluate/What-Migrates-And-What-Does-Not.md) — the complete, deliberately specific inventory.
- [**Static Code Review & Control-Flow Analysis**](1-Evaluate/MissionCriticalEmail_Migration_Software_Code_Review.md) (also as [PDF](1-Evaluate/MissionCriticalEmail_Migration_Software_Code_Review.pdf)) — the engineering review of the migration software, written to be shared with security and audit teams.
- [**Information Sheet (PDF)**](1-Evaluate/MCE-Zimbra-Migration-Information-Sheet.pdf) — a two-page overview of the offering.

## Phase Two — Get Ready

Three read-only scripts, run as the `zimbra` user on your **source** system, in this order. Each has its own usage guide.

1. [**Pre-Migration Script 1 — Fix Broken Shares On The Source System**](2-Get-Ready/Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.md) · [script](2-Get-Ready/Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh)
2. [**Pre-Migration Script 2 — Ensure All Accounts Are IMAP-Capable**](2-Get-Ready/Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.md) · [script](2-Get-Ready/Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh)
3. [**Pre-Migration Script 3 — Identify Broken Inheritance In Your Classes Of Service**](2-Get-Ready/Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.md) · [script](2-Get-Ready/Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh)

Even if you are not migrating, Script 1 is worth running periodically — every long-lived Zimbra system accumulates broken shares.

## Phase Three — Get Set

Everything that has to be in place before your licensed build is delivered.

1. [**Get Set — IMAPSYNC Server Preparation Guide**](3-Get-Set/Get-Set-IMAPSYNC-Server-Preparation-Guide.md) — provision the host that copies the mail, and avoid the classic inode exhaustion failure.
2. [**Get Set — Determine How Many Seats You Will Need To License**](3-Get-Set/Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.md) · [script](3-Get-Set/Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh) — the exact licensable mailbox count, for the whole system or for selected domains.
3. [**Get Set — Contract and Licensing Agreement Request**](3-Get-Set/Get-Set-Contract-And-Licensing-Agreement-Request.md) (also as a [fillable form](3-Get-Set/Get-Set-Contract-And-Licensing-Agreement-Request.html)) — the information we need to issue your agreement and build your binary.

## Phase Four — Go

- [**The Operator Guide**](4-Go---The-Operator-Guide/README.md) — prerequisites, the complete command reference, the two cutover workflows, and the IMAPSYNC bundle. This is the document you have open during the migration.

---

## Licensing of this repository

The four preparation scripts (Phases Two and Three) are **free and open source under the MIT License** — use them, modify them, redistribute them. Everything else in this repository (the guides, the code review, the information sheet) is copyright Mission Critical Email LLC, all rights reserved, and is provided for reference. See [`LICENSE`](LICENSE).

The migration software itself is a separately licensed, proprietary product and is **not** in this repository — Mission Critical Email delivers your build directly, licensed to your source and destination systems.

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>

*Zimbra and the Zimbra logo are trademarks of Synacor, Inc.*
