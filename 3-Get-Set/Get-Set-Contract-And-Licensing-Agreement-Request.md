<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.
-->

# Get Set — Contract and Licensing Agreement Request

Your migration build is licensed to **your** systems: it is bound to the source and destination hostnames, to the scope of the migration (the entire system, or selected domains), and to a seat count. To issue the Contract and Licensing Agreement and build your binary, Mission Critical Email needs the information below.

**Two ways to complete this:**

- Open the [**fillable form**](Get-Set-Contract-And-Licensing-Agreement-Request.html) in a browser (download the file, then open it — it is self-contained and sends nothing anywhere until you click *Email the request*), or
- copy the template below into an email.

Either way, send the completed request to **<sales@missioncriticalemail.com>**. We reply with the agreement for signature; the build follows once it is executed.

---

## 1. Scope of the migration

Choose one:

- [ ] **Entire system** — every domain on the source system is migrated.
- [ ] **Selected domains only** — list every **primary (local) domain** to be migrated, one per line. Alias domains ride along automatically and need not be listed.

  ```
  Domains to be migrated:
  1.
  2.
  3.
  ```

  > If you are migrating in waves (a multi-tenant system moved a few domains at a time), list the domains of **this** wave. Each wave is licensed and built separately.

## 2. Seat count

Run [Get Set — Determine How Many Seats You Will Need To License](Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.md) on the source — with `--domains` if you chose selected domains above — and enter the **licensable mailbox count** it reports:

```
Licensable mailbox count (from the script): ________
Seats requested (count, plus any headroom for mailboxes you expect to add before migration): ________
```

The build will not run if more mailboxes are present than were licensed.

## 3. The systems

```
Source system FQDN:        ______________________________________
Destination system FQDN:   ______________________________________
```

Give the fully-qualified hostname of the **mailbox server** on which the migration software will run on each side — the value `zmhostname` prints there. Same source and destination FQDN is normal for an in-place rebuild (you build the new server with the old server's identity and move the IP at cutover); the build supports it.

## 4. The contracting entity

```
Full legal name of the corporate entity executing the agreement:
______________________________________________________________

Physical address of that entity (street, city, state/province, postal code, country).
No PO boxes.
Street:            ______________________________________
City:              ______________________________________
State / Province:  ______________________________________
ZIP / Postal code: ______________________________________
Country:           ______________________________________
```

## 5. Contact for the agreement

```
Name:    ______________________________________
Title:   ______________________________________
Email:   ______________________________________
Phone:   ______________________________________
```

## 6. Timing (optional, helps us set the license validity window)

```
Earliest date the first dry run could start:  ____________
(The migration binary expires 60 days after this date.)
```

---

Send to **<sales@missioncriticalemail.com>** with the subject *Contract and Licensing Agreement Request*.

**Mission Critical Email LLC**
