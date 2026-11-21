# rvs-summary
The RVS Summary Dashboard for counties, states and private clients to view results in phase = complete.

## Who sees which records
Set up in the **RVS Access Admin** page (https://cusec-org.github.io/rvs-admin/). At sign-in the app checks the account:

- **CUSEC RVS Management** members (and org administrators) see **every** record, and `?edit=1` editing works as before.
- **Everyone else** sees only the read-only **views** of the RVS layer shared with their group(s): items tagged
  `RVS Records` (the admin page's wizard makes and tags them). ArcGIS enforces each view's filter, so a State
  never gets a private client's buildings. Someone in several audiences gets all their views, each record once.
  `?edit=1` opens read-only for them.
- **No view shared with the account yet:** the old read-only layer is used while it's still shared with them,
  with private clients' records (`ClientId` filled in) left out of the list. Once that layer is no longer shared
  organization-wide, they see "Your account isn't set up to see any RVS records yet."
- Reports / attachments: CUSEC admins read the reports layer; others read reports views tagged `RVS Reports`
  shared with them (none yet), else the reports view while it is still shared with them.
- The offline copy of the records is kept per account, and dropped when ArcGIS refuses access.

Settings: `CONFIG.access` in `index.html` (Management group ID, org ID, the two tags).
