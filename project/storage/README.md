# pdu_outlet_startup_config

Configures PDU outlet **startup behavior** (state-on-startup + power-on
delay) from inventory data, as part of a larger controlled
datacenter-startup-sequence effort.

This is a **working skeleton, not a finished role** — several things are
placeholders until you confirm details. Search for `TODO` in the task files.

## Data model (inventory-driven, no bundled CSV/xlsx)

- **PDUs are inventory hosts**, in a `pdus` group. Each PDU's host_vars
  define `pdu_outlet_count` (how many outlets it has). `group_vars/pdus.yml`
  sets `ansible_connection: local` since PDUs are API targets, not
  SSH-managed hosts — the role's uri calls run on the controller, hitting
  `ansible_host`/`inventory_hostname` as the API address.
- **Any host claims an outlet** by setting three vars in its own
  `host_vars`: `pdu_host` (which PDU), `pdu_outlet_id` (which outlet number),
  `pdu_startup_delay` (seconds). See `inventory_example/host_vars/esxi01.yml`
  for the pattern.
- **Unused outlets are implicit** — any outlet number in
  `1..pdu_outlet_count` that no host claims is automatically treated as
  unused: forced to `off` / `1200`. Nothing needs to be manually flagged.

This means: to onboard a new host, add a handful of lines to its
`host_vars`, same as your other roles already do — no separate file to
keep in sync, and it's a straightforward per-host diff in a PR.

## What the role does (`tasks/main.yml`)

1. Asserts `pdu_apply_confirm=true` was passed (safety gate).
2. Asserts the current PDU (`inventory_hostname`) has `pdu_outlet_count` set.
3. Scans `hostvars` for every host in inventory that claims this PDU
   (`pdu_host == inventory_hostname`), pulling their outlet_id + delay.
4. Fails loudly if two hosts claim the same outlet_id on the same PDU.
5. Builds the full `1..pdu_outlet_count` range, filling any outlet not
   claimed by a host with `off` / `1200`.
6. For each outlet, optionally GETs current settings first
   (`pdu_check_before_write: true`, the default) and skips the write if
   already correct, then PATCHes
   `rest/mbdetnrs/{{ pdu_api_version }}/powerDistributions/1/outlets/{{ id }}/settings`.

## Things you still need to confirm

1. **PDU platform / firmware** — the URL pattern looks like an Eaton ePDU G3
   style REST API, which conventionally uses `startupState`
   (`on`/`off`/`lastKnownState`) and `powerOnDelay` (seconds) on the
   `/settings` resource. Verify against the PDU's own API docs or your
   other existing PDU role's payload — don't trust the field names in this
   skeleton as-is.
2. **Auth mechanism** — copy whatever `url_username`/`url_password`, bearer
   token, or API key pattern your other PDU role already uses into the
   commented TODO blocks in `tasks/configure_outlet.yml` and
   `group_vars/pdus.yml`.
3. **HTTP verb** — used `PATCH` here; some PDU firmwares expect `PUT` with a
   full settings object instead of a partial patch. Confirm and adjust.

## How to run

```bash
ansible-playbook -i inventory_example/hosts.ini example_playbook.yml \
  -e pdu_apply_confirm=true
```

The confirmation gate is intentional — mirrors the pattern used elsewhere
in your PKI/cert automation, so this can't be run against production PDUs
by accident.

## Migrating from the Excel sheet

Treat the xlsx → inventory conversion as a **one-time, reviewed step**
rather than something the playbook parses live on every run:

1. Export/read the sheet (hostname, state, delay, and whatever tells you
   which PDU + outlet number each host is on).
2. Generate one `host_vars/<hostname>.yml` per active host with
   `pdu_host` / `pdu_outlet_id` / `pdu_startup_delay`.
3. Commit that as a normal PR — a spreadsheet typo gets caught in review,
   not during an actual PDU write.

If the mapping changes often enough that manual conversion becomes a
burden, that's a signal to eventually source this from a CMDB/dynamic
inventory instead of a spreadsheet — worth keeping in mind for later, not
a blocker now.