<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The script in this directory is
  separately MIT-licensed; this notice does not apply to it.)
-->

# Phase Three — Get Set

Everything that has to be in place before your licensed build can be issued and delivered. Three items, in order:

| # | Document | Purpose |
|---|---|---|
| 1 | [**Get Set — IMAPSYNC Server Preparation Guide**](Get-Set-IMAPSYNC-Server-Preparation-Guide.md) | Provision the host that copies the mail: sizing, placement, and the XFS cache volume that keeps a large migration from running out of inodes. |
| 2 | [**Get Set — Determine How Many Seats You Will Need To License**](Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.md) · [script](Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh) (MIT) | Run on the source to get the exact licensable mailbox count — for the entire system, or for the selected domains of a domain-scoped license. |
| 3 | [**Get Set — Contract and Licensing Agreement Request**](Get-Set-Contract-And-Licensing-Agreement-Request.md) · [fillable form](Get-Set-Contract-And-Licensing-Agreement-Request.html) | The information Mission Critical Email needs to issue your agreement and build a binary licensed to your systems. |

The build you receive is bound to the source and destination systems and the scope (entire system or selected domains) you give us in item 3, and it **will not run if more mailboxes are present than were licensed** — which is why item 2 comes first.

---

When your build arrives, continue to [Phase Four — Go](../4-Go---The-Operator-Guide/README.md).

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
