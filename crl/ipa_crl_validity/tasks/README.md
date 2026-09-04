# ipa_crl_validity

Sets the FreeIPA/Dogtag `MasterCRL` validity window on the CRL generation
master, forces a regeneration, and proves the published CRL matches intent.

## The model

```
published nextUpdate ≈ thisUpdate
                       + ca.crl.MasterCRL.autoUpdateInterval     (rebuild cadence)
                       + ca.crl.MasterCRL.nextUpdateGracePeriod  (extra validity)
```

Defaults give a 4-hour rebuild and a ~365-day `nextUpdate`.

Two failure modes this role exists to prevent:

- **`enableDailyUpdates=true`** (the Dogtag default, with `dailyUpdates=1:00`).
  The next scheduled update becomes the next daily slot, so `nextUpdate` lands
  roughly one day out no matter what the interval or grace period say.
- **Editing `CS.cfg` while `pki-tomcatd` runs.** The service serialises its
  in-memory config back over the file on shutdown, silently discarding edits.

## Usage

```yaml
- hosts: ipa_crl_master
  become: true
  gather_facts: true          # ansible_date_time and ansible_fqdn are required
  roles:
    - role: ipa_crl_validity
```

Target exactly one host. Preflight aborts if `ipa-crlgen-manage status` does
not report generation enabled.

## Tags

| Tag          | Does                                                        |
| ------------ | ----------------------------------------------------------- |
| `preflight`  | Identity, CRL-master and validity-model assertions. `always` |
| `configure`  | Drift check, backup, stop → edit → start, notify handler     |
| `regenerate` | Forced `updateCRL` call, asserts the CRL number advanced     |
| `validate`   | Reads the published CRL, asserts the window, reports state   |

Read-only check of a live CA:

```bash
ansible-playbook site.yml --tags validate
```

Regenerate without touching config:

```bash
ansible-playbook site.yml --tags regenerate -e ipa_crl_force_regenerate=true
```

## Key variables

| Variable                     | Default  | Notes                                        |
| ---------------------------- | -------- | -------------------------------------------- |
| `ipa_crl_regen_minutes`      | `240`    | Rebuild cadence = revocation latency         |
| `ipa_crl_grace_minutes`      | `525600` | Extra validity on `nextUpdate`               |
| `ipa_crl_max_regen_minutes`  | `1440`   | Preflight ceiling on the rebuild cadence     |
| `ipa_crl_clear_cache`        | `true`   | Rebuild from LDAP, not the in-memory cache   |
| `ipa_crl_force_regenerate`   | `false`  | Regenerate even when `CS.cfg` is unchanged   |
| `ipa_crl_update_timeout`     | `600`    | `updateCRL` can take minutes on a large CA   |

## Scope and limits

- **Provisioning only.** Nothing here installs a cron or timer. Routine CRL
  rebuilds are the CA's own `autoUpdateInterval`; distribution belongs to
  `crl_fetch` / `apache_crl`.
- **Not idempotent in the strict sense on first run.** The drift check runs
  `lineinfile` in check mode and reports `changed` for each setting that would
  move. That is diagnostic output, not a write.
- **`clearCRLCache=true` is slow** on a CA with a large revocation list; it
  reads the full set back from LDAP. Fine for a cutover, avoid in a loop.
- **Not validated against appliance clients.** A year-long window is legal but
  unusual, and embedded stacks sometimes cap acceptable CRL lifetimes. The
  Alcatel switch, the Safran clock and the BMCs need testing separately.
- **Freshness must be tracked separately.** With a year-long `nextUpdate`,
  "not expired" no longer implies "not stale". The validate phase warns on
  `thisUpdate` age past 3× the rebuild interval; wire the same check into
  monitoring rather than relying on consumers to complain.
