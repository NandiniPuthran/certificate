# Dashboard RBAC tests (FreeIPA → Keycloak → Dashboard)

Automates the API-level test cases from `dashboard_rbac_test_cases.xlsx`.
The Excel workbook stays the execution/sign-off record; this playbook produces a CSV
whose Result/Actual columns you paste into the Test Cases sheet.

## How the Excel and the playbook fit together

| Excel sheet        | Role in automation |
|--------------------|--------------------|
| Config             | Source of the values in `group_vars/all/main.yml` |
| Permission Matrix  | Once confirmed, each "Deny" cell becomes an entry in `rbac_api_deny_checks`, each "Allow" a candidate for `rbac_api_allow_checks` |
| Test Cases         | Paste Result/Actual from `results/rbac_results_<ts>.csv`; run the manual ones by hand |

## Coverage

| Automated | Manual (browser / log review) |
|-----------|-------------------------------|
| RBAC-01..10 (09 needs lifecycle on) | RBAC-20..24 UI parts, RBAC-30..32 |
| RBAC-20/22/24 as API allow-checks | RBAC-53 (Keycloak admin console) |
| RBAC-40..48 | RBAC-55 (idle timeout) |
| RBAC-50..52, 54 | RBAC-60, 61, 63 (audit review) |
| RBAC-62 as a deny-check | RBAC-05 effective behaviour |

## Prerequisites

1. **Direct access grants** on the Keycloak client (password grant). If the production
   dashboard client must keep them off, create a test client with the same role/group
   mappers and point `rbac_client_id` at it — but then also add an audience mapper so its
   tokens are accepted by the dashboard, and keep in mind RBAC-07/47 now test that client.
2. **The dashboard API must accept Keycloak bearer tokens.** Preflight stops the run if it
   does not, because every deny-check would otherwise "pass" with a 401.
3. Dedicated test users in FreeIPA (`t_admin`, `t_sysadmin`, `t_business`, `t_norole`,
   optional `t_multirole`, `t_nested`) in the right groups.
4. For lifecycle tests: a host in the `[ipa]` inventory group with the `ipa` CLI and a
   Kerberos ticket (or `rbac_ipa_keytab` + `rbac_ipa_principal`) allowed to change group
   membership and enable/disable users.
5. Controller can reach dashboard and Keycloak; CA at `rbac_ca_path` (FreeIPA CA by default).

## Setup

```bash
cp group_vars/all/vault.yml.example group_vars/all/vault.yml   # fill in passwords
ansible-vault encrypt group_vars/all/vault.yml
vi group_vars/all/main.yml                                     # replace every EXAMPLE value
```

Get real API paths from the browser dev tools (Network tab) while logged in as each role.

## Running

```bash
# Safe first run: read-only checks only
ansible-playbook -i inventory/hosts.ini site.yml --ask-vault-pass

# One phase at a time
ansible-playbook -i inventory/hosts.ini site.yml --tags identity
ansible-playbook -i inventory/hosts.ini site.yml --tags enforcement,token_attacks

# Write deny-checks (only against a test environment / throwaway IDs)
ansible-playbook ... -e rbac_allow_write_checks=true

# Lifecycle: modifies FreeIPA, auto-reverted in 'always' blocks. Run last, on its own.
ansible-playbook ... --tags lifecycle -e rbac_lifecycle_enabled=true

# Token expiry test (waits for the Business token to expire)
ansible-playbook ... --tags token_attacks -e rbac_expired_token_test=true
```

Tags: `identity`/`group_a`, `enforcement`/`group_d`, `token_attacks`, `session`, `lifecycle`/`group_e`.
Preflight, token acquisition and the report always run.

## Safety design

- Read-only by default. Write-method deny-checks are skipped (reported `N/A`) unless enabled:
  if RBAC is broken, a "denied" POST would really create or delete something.
- Lifecycle tests only touch users whose name starts with `rbac_test_user_prefix`, check
  their precondition first (so a revert never removes access that existed before), and
  revert in `always` even if the test fails.
- One wrong-password attempt only (RBAC-08), below lockout thresholds.
- Tokens and passwords are hidden (`rbac_no_log: true`); set false only while debugging.
- Every test records a result instead of failing mid-run; the play fails at the end if any
  test failed (`rbac_fail_on_findings`).

## Reading results

- `Pass` / `Fail` as in the Excel sheet.
- `Blocked`: the test could not be made meaningful (e.g. the admin-only probe path is not
  actually admin-only, or no foreign-client token could be obtained).
- `N/A`: skipped by a safety switch.
- 404 counts as "denied" by default. It is only meaningful if the same path returns 2xx for
  the allowed role — that is what the allow-checks are for.
- RBAC-50/52 timings are the real revocation window. A Fail there is usually a finding
  about token lifespan / session design, not a broken test.

## Known limitations

- Written and syntax-checked (YAML + Jinja), JWT logic unit-tested; not yet run against a
  live Ansible/Keycloak. Expect to adjust endpoint paths and possibly status codes.
- Needs ansible-core ≥ 2.11 (uri `ca_path`).
- `rbac_role_claim: groups` requires `rbac_roles` values to be the FreeIPA group names.
- If the realm has "Revoke Refresh Token" enabled, the RBAC-52 refresh timing is not meaningful.
