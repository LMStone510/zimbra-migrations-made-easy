<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.  (The four preparation scripts in
  this repository are separately MIT-licensed; this notice does not apply to them.)
-->

# Phase Four — Go: The Operator Guide

This is the step-by-step guide to performing the migration with the **Mission Critical Email migration software**. It assumes the first three phases are done:

- [**Phase One — Evaluate**](../1-Evaluate/): you know [what migrates and what does not](../1-Evaluate/What-Migrates-And-What-Does-Not.md).
- [**Phase Two — Get Ready**](../2-Get-Ready/): the three pre-migration scripts have been run on the source and their fixes applied.
- [**Phase Three — Get Set**](../3-Get-Set/): the IMAPSYNC server is provisioned, the seat count is known, the agreement is executed, and your licensed build has been delivered.

The migration software is a separately licensed product. It is **not** in this repository — Mission Critical Email provides your build directly. This guide tells you, in order, what to run and why it matters.

> Throughout, the binary is written as `mce-backup-restore`. Your delivered build will have a different filename; substitute its path.

---

## 1. The migration in one picture

```
SOURCE (your live Zimbra)                    DESTINATION (the new Zimbra)
─────────────────────────                    ───────────────────────────
1. Pre-Migration Scripts 1–3 (Phase Two) ──► apply fixes
2. mce-backup-restore -b … -mode pre|full
        (creates an export dir + IMAPSYNC bundle)
                    │
                    │  rsync the export to the destination Zimbra system
                    │  rsync the IMAPSYNC bundle to the imapsync server
                    ▼
                                   3. mce-backup-restore -r … -mode pre|full|post
                                      (recreates accounts, data, shares)
                                   4. IMAPSYNC bundle  ──► run on an IMAPSYNC host
                                      justfolders → premigration → postmigration
                                      (creates email folders & copies the mail blobs)
                                   5. cutover (move DNS / Elastic IP)
```

Two ways to sequence this — pick one **before** you start (Sections 5 and 6):

- **Workflow A — Full mode.** One maintenance window. Simpler. Users see empty mailboxes for the first few hours while mail catches up. Good for a few hundred users or fewer.
- **Workflow B — Pre/Post mode (recommended).** Infrastructure and the bulk of mail move *before* cutover during business hours; the cutover window is just the final delta (typically 2–4 hours). Best when downtime must be minimal.

**Always run a full dry-run cycle first, and then wipe the destination before doing a production migration!**

---

## 2. Before you begin — the Phase Two and Phase Three checklist

The migration software prompts, at the start of every backup, for confirmation that the source has been prepared. Do not answer "yes" until each of these is true:

- [ ] **Broken shares cleaned up** on the source — [Pre-Migration Script 1](../2-Get-Ready/Pre-Migration-Script-1-Fix-Broken-Shares-On-The-Source-System.md) run, its removal scripts reviewed and applied, and a re-run exits clean.
- [ ] **IMAP enabled everywhere it matters** — [Pre-Migration Script 2](../2-Get-Ready/Pre-Migration-Script-2-Ensure-All-Accounts-Are-IMAP-Capable.md) run and its provisioning file applied with `zmprov -f`.
- [ ] **CoS overrides reviewed** — [Pre-Migration Script 3](../2-Get-Ready/Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.md)'s report read, so nothing in the "will not migrate" list is a surprise during the dry run.
- [ ] **IMAPSYNC host provisioned** per the [IMAPSYNC Server Preparation Guide](../3-Get-Set/Get-Set-IMAPSYNC-Server-Preparation-Guide.md), including the XFS cache volume.
- [ ] **Seat count covered** — the licensed seat count is at least the licensable mailbox count reported by the [seat-count script](../3-Get-Set/Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.md). The binary will not run if more mailboxes are present than were licensed.
- [ ] **The build is yours** — `mce-backup-restore -license` prints the source and destination hosts, the domain scope, the seat cap, and the validity window you requested.

---

## 3. Prerequisites

Run everything as the **`zimbra`** user, on a single Zimbra **mailbox** server, inside a `screen` or `tmux` session (these runs are long); one mailbox server on each of the source and destination systems.

- **Source & destination:** a working Zimbra mailstore on each end. Do **not** pre-create Classes of Service on the destination — the binary migrates them (pre-creating them causes `_restored` duplicates). Configure the destination's MTA, anti-spam/AV, and server-wide `zimbraPublicServiceHostname/Port/Protocol` to the destination's own environment before restoring.
- **Maximum message sizes — destination must accept what the source holds.** Check that `zimbraMtaMaxMessageSize` and `zimbraFileUploadMaxSize` on the destination are **no smaller than on the source** — otherwise the `premigration` IMAPSYNC pass will likely fail to transfer the larger message blobs (and large attachment uploads can fail too). Compare with one command on each system, and raise the destination to at least the source's values:

  ```bash
  zmprov gacf zimbraFileUploadMaxSize zimbraMtaMaxMessageSize      # run on BOTH systems and compare
  zmprov mcf zimbraMtaMaxMessageSize <bytes>       # on the destination, if smaller than the source
  zmprov mcf zimbraFileUploadMaxSize <bytes>       # on the destination, if smaller than the source
  ```

- **Redo-log rollover — keep IMAPSYNC from filling the destination's disks.** Every message IMAPSYNC delivers is also written to the destination's redo log, and on Network Edition the rolled-over logs are kept in the redo-log archive directory — so a large migration writes a second copy of every blob and can fill the mailbox server's disk mid-run. Before running IMAPSYNC, on **every destination mailbox server**:

  ```bash
  zmprov mcf zimbraRedoLogDeleteOnRollover TRUE
  zmmailboxdctl restart
  ```

  With this set, redo logs are deleted at rollover instead of archived. **Revert it (set `FALSE`, restart mailboxd again) immediately before the cutover**, so the destination resumes keeping redo-log archives once it holds live mail.
- **Network Edition destinations on Zimbra 10.1+ (V3 licensing):** The licensing engine in Zimbra 10.1 takes a very strict approach to licensed features entitlements, and will block any accounts from being migrated if Classes of Service are configured such that one or more entitlements could possibly be exceeded. In practice, this happens only when the V3 license on the destination system contains Network Edition Standard seats, either exclusively or in combination with Network Edition Professional seats. Before launching the first dry-run migration, make sure that the new "default" Class of Service on the destination system as well as all of the Classes of Service on the source system, are configured to prevent you from even accidentally exceeding your entitlements for Mobile, Archiving, S/MIME, ZCO, EWS and EAS. See <https://www.missioncriticalemail.com/2025/06/09/zimbra-10-1-licensing-server-lds-best-practices/> for more detail.
- **Admin console reverse proxy (TCP 9071) — on both source and destination:** the binary runs on a single mailbox server but needs to reach all accounts in a multi-server environment, so we route this traffic through the proxy service (even on a single server). To proxy the Admin Console, run on every proxy server:

  ```bash
  /opt/zimbra/libexec/zmproxyconfig -e -w -C -H $(zmhostname) && zmproxyctl restart
  ```
- **The preparation scripts** (Phases Two and Three of this repository) have already been run; their requirements are listed in their own guides.
- **The IMAPSYNC host:** any box that can reach both servers' IMAP ports and has `imapsync` installed. `tmux` is strongly recommended. Don't run out of inodes! See: [IMAPSYNC Server Preparation Guide](../3-Get-Set/Get-Set-IMAPSYNC-Server-Preparation-Guide.md).
- **Data transfer:** the `zimbra` user has no password and its SSH keys must not be touched, so move the export with an unprivileged admin account that has an SSH key pair on both hosts and is a member of the `zimbra` group on the source.

---

## 4. The binary — complete command reference

One operation per run. You choose backup (`-b`) **or** restore (`-r`), never both. Single or double dashes are both accepted (`-mode` and `--mode` are identical).

| Switch | Side | Default | What it does |
|---|---|---|---|
| `-b <dir>` | SOURCE | — | **Backup.** Export directory. Selects the backup operation. |
| `-r <dir>` | DEST | — | **Restore.** Import directory. Selects the restore operation. |
| `-mode <pre\|full\|post\|shares>` | both | — | Phase to run (see below). Backup accepts `pre`, `full`, `post`. `shares` is restore-only. |
| `-j <N>` | both | `1` | Parallel workers. Guidance: **CPU cores − 2** (8 cores → `-j 6`). |
| `--imapsync-groups <N>` | backup `pre`/`full` | `4` | How many round-robin IMAPSYNC runner groups to generate (parallelism for the mail copy). Typical 2–10. |
| `--imapsync-only` | backup | off | Regenerate the IMAPSYNC bundle from an **existing** export, without re-running the backup. Used with `-b` and (optionally) `--imapsync-groups`; no `-mode`. |
| `-retries <N>` | restore | `3` | Retries for a failed folder or filter import (the curl-via-proxy data path). |
| `--record <dir>` | both | off | Write a JSON-lines transcript of **every external command** the binary runs to `<dir>/commands.jsonl`. A transparency/audit aid — nothing else changes. **Handle transcripts as secrets**: they capture command arguments verbatim, including credentials. |
| `-license` | either | — | Print this binary's embedded license — domain scope, source/destination hosts, seat cap, validity — and exit. Works on any machine; no Zimbra needed. The definitive answer to "which migration is this binary for?" |
| `-h` | — | — | Usage. |

**The four modes:**

- **`pre`** — infrastructure only: Classes of Service, per-domain DKIM, domains and aliases, accounts + passwords + COS assignments, external virtual accounts, conference rooms, distribution lists + members, email aliases.
- **`post`** — mailbox data (requires `pre` done): address books, calendars, briefcase, tasks, custom-folder skeletons, filters, advanced attributes, signatures, then **share grants and mountpoints** last.
- **`full`** — `pre` + `post` in one pass. The folder-skeleton and shares steps are run *separately* afterward (see Workflow A).
- **`shares`** — restore-only. Stand-alone re-run of grants-then-mountpoints. Safe to re-run anytime (already-applied grants log non-fatal and continue); uses `zmmailbox -z`, so no admin credentials needed.

**Prompts at the start of a run** (collected up front so you can walk away):

- Backup `pre`/`full`: confirm broken-share cleanup, then the **six IMAPSYNC values** — source IMAP host, dest IMAP host, source admin user/pass, dest admin user/pass. All six required; blank aborts.
- Backup `post`/`full` and restore `post`/`full`/`shares`: a Zimbra **admin user/password** (for the curl-via-proxy data path) and the proxy host.
- **Why we ask for two (potentially different) Admin accounts when doing exports.** You can use a 2FA-enabled Admin account for the IMAPSYNC server; just be sure to create an application-specific password and use that password. 2FA-enabled Admin accounts don't work for the work that goes through the Proxy (it's an auth token thing), so that Admin account cannot have 2FA enabled. It's not uncommon that the IMAPSYNC server is hosted someplace else, so the two sets of accounts enable you to use a protected/throwaway Admin account/password for the IMAPSYNCs, and not expose TCP port 9071 to the public Internet.

**Running unattended** (stdin not a TTY): the binary reads these env vars instead of prompting (interactive runs ignore them):

```bash
# curl-via-proxy data (backup post/full, restore post/full/shares)
ZIMBRA_ADMIN_USER   ZIMBRA_ADMIN_PASS   ZIMBRA_PROXY_HOST

# IMAPSYNC credentials (backup pre/full)
IMAP_HOST1  IMAP_HOST2  IMAP_AUTHUSER1  IMAP_PASSWORD1  IMAP_AUTHUSER2  IMAP_PASSWORD2
```

---

## 5. Workflow A — Full mode

Complete Sections 2–3 first, and back up with `-mode full`.

```bash
# (source) — backup example command
time nice -n 19 ionice -c2 -n7 \
  mce-backup-restore -b /opt/zimbra/export -j 6 -mode full
```

1. **Transfer** the export to the destination (rsync to staging, then move into `/opt/zimbra/import`, `chown -R zimbra:zimbra`, `chmod 700`).
2. **Restore (full):**
   ```bash
   time nice -n 19 ionice -c2 -n7 \
     mce-backup-restore -r /opt/zimbra/import -j 6 -mode full
   ```
   Recreates accounts and mailbox data. Does **not** restore shares or copy mail. Typically 1–3 hours.
3. **Place the IMAPSYNC bundle.** Copy the entire `imapsync/imapsync-multi/` directory (it is self-contained) to the IMAPSYNC host.
4. **Folder skeleton:** `bash launch-all-tmux.sh justfolders` — creates every custom folder on the destination (no mail). Minutes. Required before shares.
5. **Cutover:** block user access to the source (keep the IMAPSYNC host's access), move DNS / the Elastic IP to the destination.
6. **Shares:**
   ```bash
   mce-backup-restore -r /opt/zimbra/import -j 6 -mode shares
   ```
   All folders exist now (step 4), so every grant succeeds. 5–20 minutes.
7. **Mail:** `bash launch-all-tmux.sh premigration` — the long blob copy (hours to days). In Workflow A this is the single post-cutover mail run.
8. **Re-arm enforcement** (after the mail run has finished and its per-user logs are clean):
   ```bash
   zmprov -f /opt/zimbra/import/deferred_quotas.zmp
   zmprov -f /opt/zimbra/import/deferred_account_statuses.zmp   # only if the restore wrote one
   ```
   Every account was migrated with an *unlimited* mailbox quota so nothing could fail over-quota mid-copy; the first batch restores the source quotas (per-account overrides where they existed, COS inheritance everywhere else). The second re-applies locked/closed account statuses, which the restore also never sets live because they block IMAP authentication.
9. **Verify.**

---

## 6. Workflow B — Pre/Post mode (recommended)

Complete Sections 2–3 first, and back up with `-mode pre`.

**Phase 1 — before cutover (days ahead, business hours):**

1. Back up the source with `-mode pre`; transfer the export to the destination.
2. **Restore (pre):** `mce-backup-restore -r /opt/zimbra/import -j 6 -mode pre` — infrastructure only.
3. **Folder skeleton:** `bash launch-all-tmux.sh justfolders`.
4. **Bulk mail:** `bash launch-all-tmux.sh premigration` — re-run as often as you like over the coming days; each run moves only what changed.

**Phase 2 — cutover night (short window):**

5. Block user access to the source.
6. Back up the source with `-mode post`; transfer.
7. **Restore (post):** `mce-backup-restore -r /opt/zimbra/import -j 6 -mode post` — mailbox data **and** shares in one pass (the folders already exist from Phase 1, so shares restore inline; no separate shares step needed).
8. **Final delta:** `bash launch-all-tmux.sh postmigration` — picks up the last few days of mail (see Section 7). Move DNS / the Elastic IP.
9. **Re-arm enforcement** (after the final delta has finished):
   ```bash
   zmprov -f /opt/zimbra/import/deferred_quotas.zmp
   zmprov -f /opt/zimbra/import/deferred_account_statuses.zmp   # only if the restore wrote one
   ```
   Quotas were suspended (unlimited) on every account from the pre restore onward, so neither the bulk passes nor the delta could fail over-quota; this restores the source quotas. Until you run it, the admin console shows every account as unlimited — that is expected, not a missed migration.
10. **Verify.**

> **Same source/destination FQDN is normal** for an NE→FOSS migration or DR: you build the new server with the old server's identity and move the AWS Elastic IP at cutover, so the hostname never changes. Your licensed binary supports this.

---

## 7. The IMAPSYNC bundle — three phases

**The bundle does two jobs beyond copying mail**, and both are why a Zimbra migration that "just copies mailboxes" gets shared folders wrong:

1. **It builds your custom folder tree.** The binary creates each account and its default Zimbra folders (`/Inbox`, `/Sent`, …), but *custom* mail folders such as `/Inbox/Projects` are created by IMAPSYNC's `justfolders` phase. That is also why `justfolders` must run before shares are restored — a grant needs its folder to already exist.
2. **It refuses to re-copy shared folders.** When a folder is shared to a user it appears in their mailbox as a *mountpoint*, not a real folder. Left alone, IMAPSYNC would copy it as a real folder and duplicate the owner's mail into the grantee's account. The binary auto-generates a per-user folder-exclude list (from each user's live folder view at backup time) so every IMAPSYNC phase skips those mountpoints — the real share is recreated by the restore instead.

The backup generates this self-contained bundle under `<export>/imapsync/imapsync-multi/`. On the IMAPSYNC host you run three phases, **in this order**:

```bash
bash launch-all-tmux.sh justfolders     # Phase 1 — folder skeleton (minutes)
bash launch-all-tmux.sh premigration    # Phase 2 — bulk mail copy (hours–days)
bash launch-all-tmux.sh postmigration   # Phase 3 — final delta (minutes; Workflow B only)
```

Each invocation fans out one detached `tmux` session per group and writes a per-group log. **Always pass the phase name** — the launcher never guesses, because `premigration` includes imapsync's destructive `--delete2` and the wrong phase at the wrong time can lose data.

- **Parallelism** is set at backup time with `--imapsync-groups N` (default 4). Use the default unless the migration is small (< ~50 GB total mail) or the source is resource-constrained — then start at 1–2 and watch source load. To change the count after a backup, regenerate the bundle with `mce-backup-restore -b <export> --imapsync-only --imapsync-groups N`.
- **Postmigration window:** the final delta defaults to the last 7 days. Size it to the full span from when the *bulk* pass started to when this delta finishes, plus a day or two. Override per run:
  ```bash
  bash launch-all-tmux.sh postmigration --maxage 10
  ```
- **Site-specific imapsync flags.** Every generated group runner has an empty `EXTRA_OPTS` array near the top; any imapsync options you add there apply to every account, in every phase. Use it for quirks specific to your environment — the classic example is the destination rejecting messages at IMAP `APPEND` time because the source stored nonstandard message flags (the fix is a `--regexflag` rule that strips or maps the offending flags); other candidates are `--maxsize` to defer oversized messages, or extra `--exclude` patterns.
- **Repeated bulk passes are cheap:** the `premigration` and `postmigration` phases keep a persistent UID cache on the IMAPSYNC host, so reruns skip already-transferred messages without re-fetching headers. The cache needs a properly provisioned directory (inodes, not gigabytes) — set it up once per the [IMAPSYNC Server Preparation Guide](../3-Get-Set/Get-Set-IMAPSYNC-Server-Preparation-Guide.md). On a rerun, each account's summary should show nearly everything as *skipped*; a near-zero skipped count means the cache isn't being found.
- **No tmux?** Run each group directly in its own terminal: `bash imapsync-group1.sh justfolders` (repeat per group, then move to the next phase). The launcher is a major convenience; tmux is strongly recommended!

---

## 8. Licensing & support

The migration binary is licensed per customer by Mission Critical Email and is bound to your server identity; the four preparation scripts in this repository are MIT-licensed (see [`LICENSE`](../LICENSE)).

**Staged (multi-wave) migrations.** Hosting a multi-tenant Zimbra system and moving it a few domains at a time? Each wave gets its own binary, licensed for exactly the domains in that wave — the binary backs up and restores *only* those domains (other tenants' accounts, DKIM signing keys, and settings never travel in that wave's backup), refuses backups containing anything outside its scope, and identifies itself with `-license`. Cross-wave shares and aliases are handled: they connect automatically once their counterpart domain arrives in a later wave. Ask us about wave licensing when you order.

To size your license before you buy, run the [seat-count script](../3-Get-Set/Get-Set-Determine-How-Many-Seats-You-Will-Need-To-License.md) on your source — it reports the exact mailbox count the migration will license — and send us the [Contract and Licensing Agreement Request](../3-Get-Set/Get-Set-Contract-And-Licensing-Agreement-Request.md). For a license, a build, or migration help:

**Mission Critical Email LLC** — sales@missioncriticalemail.com
