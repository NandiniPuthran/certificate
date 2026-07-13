# jenkins_monitor_svc_setup

Ansible role that provisions a Jenkins **read-only monitoring service account**
end-to-end, for instances secured by Keycloak/OIDC (or any realm where local
account creation is restricted, and API tokens are the only viable
non-interactive credential).

## What it does (three phases, each toggleable)

1. **`generate_token`** — creates/renews a Jenkins API token for the target
   user via the Script Console (`/scriptText`), using `User.getById(id, true)`
   so no interactive login or local-account creation is required.
2. **`assign_permissions`** — creates (if missing) and binds two
   Role-Strategy-plugin roles:
   - a **global role** granting `Overall/Read` (baseline, required for any
     API access at all)
   - an **item/project role** granting `Job/Read`, scoped by a regex pattern
     (defaults to `.*` — all jobs; narrow this if you want tighter scope)
3. **`validate_readonly`** — with zero prior knowledge of job names:
   - discovers every job visible to the account
   - confirms console logs are actually readable (`consoleText`)
   - confirms the `/configure` page is **blocked** (non-destructive way to
     prove no write/`Job.Configure` access leaked in)
   - fails the play if either check comes back wrong

## Requirements

- Jenkins Role Strategy plugin (`com.michelin.cio.hudson.plugins.rolestrategy`)
- An admin-level API token for `jenkins_admin_user`, used only to drive the
  Script Console — never used by the monitoring account itself
- Ansible 2.14+

## Usage

```yaml
- name: Provision monitor-svc token + read-only role + validate
  include_role:
    name: jenkins_monitor_svc_setup
  vars:
    jenkins_url: "https://jenkins.internal"
    jenkins_admin_user: "admin"
    jenkins_admin_token: "{{ vault_jenkins_admin_token }}"
    jenkins_monitor_user: "monitor-svc"
    jenkins_item_role_pattern: ".*"
```

See `example_playbook.yml` for a complete runnable example.

## Re-running / idempotency

- Token generation revokes any existing token with the same `jenkins_token_name`
  and re-creates it — safe to re-run, but note this **invalidates the old
  token value** every time. If you only need to re-validate, set
  `jenkins_generate_token: false` and pass the existing token via
  `-e jenkins_monitor_api_token=...` instead.
- Role/assignment creation checks for existing roles by name before creating,
  so re-running won't duplicate roles.

## Safety notes

- The write-access check deliberately uses `GET .../configure` (renders the
  edit page) rather than `POST .../config.xml` or `POST .../build` — this
  avoids any risk of an XML-parser exception or an actually-triggered build
  if permissions turn out to be mis-scoped. A `403` on `GET /configure` is
  just as strong a "no write access" signal without any destructive side
  effect.
- `no_log: true` is applied to every task that touches the token value, so it
  never lands in playbook output or CI logs.
- `jenkins_persist_token` defaults to `false` — the token fact only lives in
  memory for the play unless you explicitly opt in to writing it to disk
  (and even then, only under `0600`). Prefer piping it into your actual
  secrets manager in a follow-up task instead of leaving it on disk.

## Role/permission scope caveat

`jenkins_item_role_pattern: ".*"` grants read visibility across **every**
job/folder on the instance. If your Jenkins security policy calls for
narrower scope (e.g. only an `infra/` or `monitoring/` folder tree), set
this to a tighter regex, e.g. `^infra/.*`, before running.
