# crl_generate

Give it a CA certificate and that CA's private key, and it produces a CRL and
copies it — with the CA certificate — to the hosts that need them.

The private key stays on the Ansible control node. Signing happens there, once
per run. Only the finished CRL and the CA certificate go to managed hosts.

---

## What it is for

Anywhere an application reads a CRL from disk and you own the CA that issued
the certificates. The original case was an initial rollout: certificates come
from a shared local CA, revocation checking is already enforcing, and the
application needs `ipa-ca.pem` and a CRL present before it will start. Later,
IPA takes over and `crl_fetch` replaces both files.

It works equally well as the permanent authority for a CRL path — that's what
`crl_generate_placeholder_only` switches between.

---

## The idempotency problem, and how it's handled

`openssl ca -gencrl` produces a **different file every time** — new
`lastUpdate`, incremented CRL number, different signature. Regenerating on
every run would push a changed file to every host on every run, and trip any
service-reload handler along with it.

So the role checks first, on the control node, and only regenerates when it
actually needs to. It regenerates when the CRL is:

| Verdict | Meaning |
|---|---|
| `absent` / `empty` | no artifact yet |
| `unparseable` | file is corrupt |
| `signature-mismatch` | signed by a different CA than the one you passed |
| `issuer-mismatch` | issuer name doesn't match the CA subject |
| `expiring N` | fewer than `crl_generate_renew_before_days` left |

Anything else reports `ok N` and generation is skipped entirely. A second run
changes nothing, on the control node or on any host.

`crl_generate_force: true` overrides this when you want a fresh CRL.

Tested branches: valid 10-year CRL → `ok 3649`; missing → `absent`; zero-byte →
`empty`; garbage → `unparseable`; CRL from another CA → `signature-mismatch`;
30-day CRL with a 90-day threshold → `expiring 30`, with a 7-day threshold →
`ok 30`. Repeat runs leave the artifact byte-identical.

---

## Expiry

`crl_generate_days` is yours to set because you hold the key. The default is
3650 (ten years), which is the practical answer to "it must not expire on us".

I'd avoid genuinely omitting the `nextUpdate` field, even though the encoding
permits it: RFC 5280 says conforming CAs must include it, and OpenSSL, GnuTLS
and NSS don't agree on how to treat its absence. A ten-year date gets the same
result without depending on that. The role does handle a CRL with no
`nextUpdate` if you produce one elsewhere — it reports `ok non-expiring`.

---

## Revocation

The generated CRL lists nothing. If you ever revoke a certificate from this CA,
record it in `{{ crl_generate_state_dir }}/index.txt` the normal
`openssl ca -revoke` way and run with `crl_generate_force: true`. The state
directory holds `index.txt` and `crlnumber`, and is never reset once created,
so CRL numbers stay monotonic — some validators reject a CRL numbered lower
than one they've already seen. Commit that directory if more than one control
node runs this.

---

## Key handling

- The key path is read on the control node only. It is never copied to a host.
- If the key is passphrase-protected, put the passphrase in
  `crl_generate_ca_key_passphrase` and vault it. It is passed to `openssl`
  through the environment rather than the command line, so it does not appear
  in the control node's process list, and the signing task runs with
  `no_log: true`.
- The key file itself should be vaulted or kept outside the repo.

---

## Variables

| Variable | Default | Notes |
|---|---|---|
| `crl_generate_name` | `local-ca` | label; separates state and artifacts per CA |
| `crl_generate_ca_cert` | — | **required**, control node path |
| `crl_generate_ca_key` | — | **required**, control node path |
| `crl_generate_ca_key_passphrase` | `""` | vault this if the key is encrypted |
| `crl_generate_days` | `3650` | CRL validity |
| `crl_generate_renew_before_days` | `90` | regenerate below this many days left |
| `crl_generate_force` | `false` | regenerate unconditionally |
| `crl_generate_digest` | `sha256` | signature digest |
| `crl_generate_artifact_dir` | `{{ playbook_dir }}/generated/crl` | generated CRL lands here |
| `crl_generate_state_dir` | `{{ playbook_dir }}/generated/crl-state/<name>` | `index.txt`, `crlnumber` |
| `crl_generate_crl_dest` | `/etc/ssl/crl/MasterCRL.pem` | PEM destination on hosts |
| `crl_generate_crl_der_dest` | `""` | optional DER copy |
| `crl_generate_ca_dest` | `/etc/ssl/certs/ipa-ca.pem` | `""` to skip |
| `crl_generate_placeholder_only` | `true` | see below |
| `crl_generate_marker` | `/etc/ssl/.crl-from-local-ca` | `""` to disable |
| `crl_generate_reload_services` | `[]` | reloaded when a file changes |
| `crl_generate_owner` / `_group` | `root` | |
| `crl_generate_file_mode` | `0644` | world-readable on purpose |
| `crl_generate_validate` | `true` | verify and report after generating |

**`crl_generate_placeholder_only`**

- `true` — write only when the destination is missing or empty. Use when
  something else owns the path later (`crl_fetch`, IPA enrolment). This is the
  initial-rollout case, and it means the role can be left in the playbook
  permanently without ever fighting the role that replaces it.
- `false` — keep the destination in sync with the generated CRL. Use when this
  role is the authority for that path.

The marker file lists whichever destinations still hold exactly what this role
wrote, and is removed once nothing does. With `placeholder_only: true` that
gives you something to alert on: its presence means the host is still on local
CA data rather than live data.

---

## Usage

Initial rollout, local CA, handing over to `crl_fetch` later:

```yaml
- hosts: crl_clients
  become: true
  roles:
    - role: crl_generate
      vars:
        crl_generate_name: site-local-ca
        crl_generate_ca_cert: "{{ playbook_dir }}/pki/local-ca.pem"
        crl_generate_ca_key: "{{ playbook_dir }}/pki/local-ca.key"   # vaulted
        crl_generate_ca_dest: /etc/ssl/certs/ipa-ca.pem
        crl_generate_crl_dest: /etc/ssl/crl/MasterCRL.pem
        crl_generate_days: 3650
        crl_generate_placeholder_only: true
```

More than one CA in the same playbook — include it twice with different
`crl_generate_name` values, so the state directories don't collide:

```yaml
- ansible.builtin.include_role:
    name: crl_generate
  vars:
    crl_generate_name: internal-ca
    crl_generate_ca_cert: "{{ playbook_dir }}/pki/internal.pem"
    crl_generate_ca_key: "{{ playbook_dir }}/pki/internal.key"
    crl_generate_crl_dest: /etc/ssl/crl/internal.crl.pem
    crl_generate_ca_dest: ""
```

## Tags

`crl_generate`, `crl_generate_assert`, `crl_generate_inspect`,
`crl_generate_generate`, `crl_generate_verify`, `crl_generate_distribute`

## Notes

- **Check mode.** `--check` can't create the CRL. If no usable artifact exists
  yet the role fails with a clear message; run once for real, then check mode
  works normally.
- **Air-gapped.** Only `openssl` and `date` are used, both already present. No
  extra collections needed. `community.crypto.x509_crl` would be a tidier
  implementation if you're willing to vendor that collection.
- **`playbook_dir` must be writable** for the default artifact and state paths.
  Point them elsewhere if it isn't.
- **Reload handler.** Off by default. Reload semantics differ per service —
  apache2 and haproxy pick up a new CRL on reload, rsyslog needs a restart.
  Confirm before listing anything, especially across 500 hosts.
