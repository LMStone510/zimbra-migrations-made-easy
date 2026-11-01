<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The script it describes is MIT-licensed.)
-->

# Get Set — Determine How Many Seats You Will Need To License

**Script:** [`Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh`](Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh) (MIT License)
**Run as:** the `zimbra` user, on any mailstore of the **source** system. Needs only `zmprov`.
**Changes anything?** No. It reports a count and writes a report file.

## What it does

The migration software counts mailboxes against your license. This script **reproduces that enumeration exactly**, so you can license precisely the right quantity — no guesswork, no over- or under-buying:

1. `zmprov -l gaa` — every provisioned account in the deployment (LDAP mode, fast; no per-mailbox probing).
2. Drop Zimbra **system accounts** — `galsync`, `ham.`, `spam.`, `virus-quarantine`.
3. Subtract **external virtual accounts** (`zimbraIsExternalVirtualAccount=TRUE`) — the stubs Zimbra creates when a folder is shared to an outside address; they hold no mail.

The survivors are the **licensable mailbox count**. The script prints the full breakdown (total → system excluded → external excluded → licensable) and writes a report plus the licensable account list for your records.

## Entire system, or selected domains?

`zmprov -l gaa` is deployment-wide (LDAP is global), so run on any one mailstore it counts the **whole deployment**. That is the number to license for migrating the **entire system**:

```bash
bash Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh --output-dir /tmp
```

For a **domain-scoped license** — a multi-tenant system moved one wave of domains at a time, or a migration of only some of your domains — pass the domains in scope. Only accounts whose *primary* address is in one of those domains are counted, which is exactly the scoping a domain-scoped build enforces:

```bash
bash Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.sh --domains "example.com, example.org" --output-dir /tmp
```

List the **primary (local) domains**; alias domains ride along automatically and never change the count (every account is counted once, by its primary address). The same list of domains goes on your [Contract and Licensing Agreement Request](Get-Set-Contract-And-Licensing-Agreement-Request.md).

Options:

| Option | Default | Purpose |
|---|---|---|
| `--domains LIST` | whole deployment | Count only these primary domains (comma- or space-separated; repeatable). |
| `--output-dir DIR` | `/tmp` | Directory for the auto-named report, `mce-mailbox-license-count_<timestamp>.txt`. |
| `-o FILE` | | Full path for the report file (overrides `--output-dir`). |
| `-h`, `--help` | | Usage. |

Exit codes: `0` count produced · `2` setup or environment error · `3` no accounts found.

## How many seats to request

License the licensable count the script reports — and consider a few extra seats if you expect to add mailboxes between now and the migration. **The migration software will not run if more mailboxes are present than were licensed.** Volume discounts apply above 2,500 mailboxes.

Next: [Get Set — Contract and Licensing Agreement Request](Get-Set-Contract-And-Licensing-Agreement-Request.md).

---

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
