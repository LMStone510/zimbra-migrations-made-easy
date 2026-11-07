<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.
-->

# Phase One — Evaluate

This phase answers two questions for a prospective customer:

1. **Is this software the right way to do my migration?**
2. **Can my security and audit teams be confident that it is well-tested and data-safe?**

If the answer to both is yes, move on to [Phase Two — Get Ready](../2-Get-Ready/).

> The authoritative overview of the offering is the product page at <https://www.missioncriticalemail.com/zimbra-migrations-made-easy/>. This directory carries the material that page points to: the full inventory of what migrates, the engineering code review, and the Information Sheet.

---

## 1. Is it the right tool? — reach for our migration software when:

1. A Rolling Upgrade or an rsync migration isn't appropriate — for example when S3 is present on the source system.
2. You bought an NE license and need to migrate from a FOSS system.
3. You're a Zimbra BSP partner onboarding a new customer's to-be-retired self-hosted Zimbra system — domain-scoped licensing migrates one wave of domains at a time onto the same destination farm.
4. You need users' email data in place before the actual cutover.

**Any version 8.8.15 or higher. Any OS. Any Zimbra edition.** FOSS ↔ Network Edition in either direction, a different operating system, and a different number of servers — all in the same move. A typical example: Zimbra 9.0.0 FOSS on Ubuntu 18 → Zimbra 10.1 Network Edition on Rocky Linux 9.

### How it works — a two-path architecture

```
SOURCE SYSTEM                MCE MIGRATION ENGINE               DESTINATION SYSTEM
Existing Zimbra   ──READ──►  PROVISIONING PATH    ──WRITE──►   New Zimbra
Any version, any OS          accounts · structure · settings    New version, new OS
FOSS or Network Edition      IMAPSYNC PATH                      New server topology
Any number of servers        all mail, message-for-message
LEFT INTACT
```

Provisioning recreates the full system — users, structure, and settings — while IMAPSYNC moves every message in parallel. **The source is read, never altered**, so you always have a clean rollback.

### What migrates

A whole Zimbra system — recreated, not approximated. The complete inventory, including the short list of what deliberately does **not** migrate, is in [What Migrates — And What Does Not](What-Migrates-And-What-Does-Not.md). On a migration, knowing *exactly* what carries over is what lets you and your security team sign off with confidence.

---

## 2. Is it well-tested and data-safe?

### The code review — written to be shared with your security and audit teams

[**Static Code Review & Control-Flow Analysis**](MissionCriticalEmail_Migration_Software_Code_Review.md) — also available as a [PDF](MissionCriticalEmail_Migration_Software_Code_Review.pdf).

It documents the migration software's architecture, control flow, error handling, and the results of the standard Go analysis and testing tooling. The properties that matter most to a reviewer, in brief:

- **Non-destructive by construction.** Backup is read-only against the source: it never modifies source mail, LDAP, or configuration. Restore only provisions and adds to the destination. The only destructive mailbox operations live in the separately-run IMAPSYNC tooling, gated by explicit, phase-specific flags.
- **No third-party code.** The binary is built from the Go standard library only — there is no dependency tree to scan and no supply-chain surface. It is a single statically-linked executable with no runtime dependencies.
- **Fully auditable.** Every external command the binary runs flows through one choke point. With `--record` enabled, that choke point writes a complete machine-readable transcript, so a reviewer can confirm exactly what the binary does on their systems.
- **Tested the way migration tooling should be.** An automated suite of 218 tests, 13 native fuzz targets over every parser of external data, static analysis with `go vet` and `staticcheck` (clean), an equivalence harness that compares the binary's behaviour against the field-proven reference implementation, and ongoing field validation on real customer cutovers.
- **Least privilege.** Runs as the unprivileged `zimbra` user, needs no root, installs nothing into Zimbra, and its only network egress is to the reverse proxy of the two systems being migrated — no telemetry, no external calls.

The review states its own scope and limitations explicitly (Section 2 of the document), so your reviewers know what it does and does not cover.

### We are committed to Open Source

Every preparation script a migration needs — all four of them — is published in this repository under the **MIT License**:

| Script | Phase |
|---|---|
| [Pre-Migration Script 1 — Fix Broken Shares On The Source System](../2-Get-Ready/Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.sh) | Get Ready |
| [Pre-Migration Script 2 — Ensure All Accounts Are IMAP-Capable](../2-Get-Ready/Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.sh) | Get Ready |
| [Pre-Migration Script 3 — Identify Broken Inheritance In Your Classes Of Service](../2-Get-Ready/Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.sh) | Get Ready |
| [Get Set — Determine How Many Seats You Will Need To License](../3-Get-Set/Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh) | Get Set |

They are useful on their own — Script 1 in particular is worth running periodically on any long-lived Zimbra system — and they are read-only: each one produces a report, and where a fix is needed, a provisioning file or removal script that *you* review and apply. None of them changes your system by itself.

The migration software itself is proprietary and is delivered directly to licensed customers; see the [Information Sheet](MCE-Zimbra-Migration-Information-Sheet.pdf) for the two-page overview.

---

## 3. Licensing and next steps

Licensing is per migration plus per mailbox migrated, with volume discounts above 2,500 mailboxes; current pricing is published on the [product page](https://www.missioncriticalemail.com/zimbra-migrations-made-easy/). Domain-scoped (per-wave) licenses are available for multi-tenant systems that move a few domains at a time.

When you are ready, continue to [Phase Two — Get Ready](../2-Get-Ready/). To talk it through first:

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
