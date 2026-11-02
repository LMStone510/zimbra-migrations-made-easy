<!--
  Copyright (c) 2026 Mission Critical Email LLC.  All rights reserved.
  This document is proprietary.  It may NOT be copied, modified, redistributed,
  or republished, in whole or in part, without the prior written permission of
  Mission Critical Email LLC.  See LICENSE.
-->

# What Migrates — And What Does Not

*Phase One — Evaluate.* This is the complete inventory of what the Mission Critical Email migration software carries from a source Zimbra system to the destination, and the short list of what it deliberately leaves behind. It is written for the people who have to sign off: the migration operator, and the security team.

The migration software migrates your Zimbra **provisioning and mailbox data**; the IMAPSYNC bundle it generates creates **email folders** and migrates the **mail blobs**. Together they move a complete system — accounts, structure, sharing, and content — from source to destination. The list below is deliberately specific: on a migration, knowing *exactly* what carries over (and what doesn't) is what lets you — and your security team — sign off with confidence.

**Infrastructure**

- **Classes of Service (COS)**, with the full inheritance chain preserved. A COS whose name already exists on the destination (`default` always does) is restored as `<name>_restored` rather than overwriting it, and every domain and account that used it on the source is bound to that restored copy — so your customizations to `default` (quota, features, password policy) follow your users rather than being silently replaced by the destination's out-of-the-box settings.
- **Local domains and domain aliases**, and each domain's default COS.
- **Per-domain DKIM signing keys** for domains configured with DKIM — so signed mail keeps verifying at cutover with no DNS change.
- **Distribution lists**, their members, and DL aliases.
- **Resource accounts** (e.g. Conference Rooms).
- **External (virtual) accounts** — the off-Zimbra-domain users to whom internal folders or resources have been shared.

**User accounts** — each account is recreated with:

- Password.
- Display Name / Given Name / Surname.
- Email aliases.
- Work Title, Company.
- Complete address (street, city, state, ZIP).
- Telephone number.
- Spell-check "words to ignore".
- Personal Amavis block / allow lists.
- Trusted Senders list.
- Preferred "From" address and display name.
- Explicit per-account COS assignment.
- All account-level Zimbra Access Control Entries (ACEs).
- All Sieve scripts (filters).
- `zimbraId`.
- **Two-factor authentication (2FA)** — each user's TOTP shared secret, backup (scratch) codes, app-specific passwords, and enabled state, carried over verbatim so users keep their existing authenticator with **no re-enrollment**. Trusted-device tokens are not carried (users simply re-trust their devices). On a FOSS destination, install a 2FA add-on (e.g. [maldua](https://github.com/maldua-suite/zimbra-maldua-2fa)) to manage and enforce these settings.

**User mailbox content**

- **All address books** — every top-level contact folder *and* its subfolders, not just `/Contacts`; each exported and imported individually, with names preserved.
- **All calendars** — every top-level calendar folder *and* its subfolders, not just `/Calendar`; each exported and imported individually, with names preserved.
- **Briefcase** and **Tasks**.
- **Email signatures**.
- **Out-of-office auto-replies** — each user's vacation message (including multi-line and international text), external-sender handling, and the enabled state with its from/until dates, applied so auto-replies re-arm as cutover-day mail starts flowing. An expired vacation window stays off. *One footnote:* Zimbra's "reply once per sender" memory starts fresh on the new server, so a sender who was auto-replied to before cutover may receive one repeat auto-reply after the cutover.
- **RSS/Atom feed folders and ICS-subscribed calendars** — recreated with their subscription URLs *before* the folder-skeleton pass, so the destination keeps polling the feed while IMAPSYNC fills in the accumulated history. A migration that "just copies mailboxes" leaves these as dead folders that silently never update again.
- **Folder shares** — outgoing grants and incoming mountpoints, recreated in the correct dependency order — including shares accepted under an account's *alias* identity, and same-named shares disambiguated automatically.

**Mail**

- All mail messages, via the auto-generated IMAPSYNC bundle (see the [IMAPSYNC bundle section of the Operator Guide](../4-Go---The-Operator-Guide/README.md#7-the-imapsync-bundle--three-phases)).

**What does not migrate** (by design):
- **Mail blobs travel via IMAPSYNC, not the provisioning path** — that is the architecture, not a gap (the [Operator Guide](../4-Go---The-Operator-Guide/README.md#7-the-imapsync-bundle--three-phases) explains why this is an advantage).
- GALsync accounts.
- ActiveSync/mobile, S/MIME, ZCO/EWS, and NE Backup settings are skipped automatically on a Network Edition → FOSS migration (they have no native FOSS equivalent and would otherwise fail account creation). *(Two-factor authentication is the exception — it **is** migrated; see "What migrates" above.)*
- Custom `localconfig` values, MTA settings, SSL certificates, branding/theming, and external auth integrations (Okta/JumpCloud/AD) — these belong to the destination's own build.
- **External domain authentication** (`zimbraAuthMech` = `ad`/`ldap`/`custom` — domains that authenticate users against Active Directory or another external LDAP directory). The `zimbraAuthLdap*` configuration names *your* directory infrastructure (servers, bind credentials, TLS trust) and must be pointed at deliberately, so it is not carried: every domain is recreated on the destination with internal (`zimbra`) authentication. The backup **warns for each external-auth domain it finds** and records them in `external_auth_domains.txt`; the restore repeats the list. **Reconfigure external authentication on the destination before users log in** — until you do, those users' directory passwords will not work there.
- Domain-level ACEs (account-level ACEs *do* migrate); and the admin-role flags themselves (`zimbraIsAdminAccount` / `zimbraIsDelegatedAdminAccount`) are intentionally not set on the destination — admins are exported for reference and reconstructed deliberately.
- Per-domain public-service hostname/port/protocol (you set those on the destination, since they often change as a result of the migration).
- **Account-level overrides of CoS-managed policy attributes that *loosen* policy, plus override cruft.** Those attributes belong at the Class of Service level, and account-level values are often forgotten one-off workarounds that silently exempt an account from its COS forever. The migration handles the class in tiers: overrides that **tighten security or encode operational policy** (password locks, forwarding-permission policy, per-seat feature grants like EWS and S/MIME, quotas, session lifetimes, 2FA-required/available = TRUE) **do migrate** — dropping those would silently weaken the destination — and every one applied is recorded by the restore in `applied_policy_overrides.txt`, with per-account **quota** overrides called out separately (they interact with COS-keyed billing plans). **All mailbox quotas on the destination are suspended (unlimited) from the pre/full restore until you re-arm them after the final IMAPSYNC pass** — the "re-arm enforcement" step in each Operator Guide workflow — so no mail copy can fail over-quota. Overrides that **loosen** policy (e.g. a 2FA exemption on an application account) and provisioning leftovers are deliberately **not** migrated: they surface during your dry-run testing, where they can be fixed properly (that application account belongs on an **app-specific password**, which the migration *does* carry — not on a 2FA exemption); [Pre-Migration Script 3](../2-Get-Ready/Pre-Migration-Script-3-Identify-Broken-Inheritance-In-Your-Classes-Of-Service.md) lists them before you start. *(User preferences — `zimbraPref*` — are not policy and migrate normally, as do account status and mail forwarding.)*

---

**Mission Critical Email LLC** — <sales@missioncriticalemail.com>
