# apache_crl

Serves the published CRL over plain HTTP from server1. Runs **after**
`crl_fetch` — it serves the directory, it does not create or fill it.

## Two audiences, one vhost

| Requests as | Who | Why |
|---|---|---|
| `ipa-ca.example.com` | certificates' CDP, browsers | site B DNS resolves it here |
| `{{ ansible_fqdn }}` | the 500-host fleet | `crl_fetch` pulls from this name |

The `ServerAlias` is not cosmetic. The fleet deliberately fetches from server1's
**own** hostname rather than the CDP alias, so distribution works whether or not
the DNS override has been cut over — the two changes stay independently
reversible. Drop the alias and 500 hosts stop updating while the CDP still
appears healthy.

## Layout

```
apache_crl/
  defaults/main.yml            11 variables
  handlers/main.yml
  meta/main.yml
  tasks/main.yml               → configure, then verify
  tasks/configure.yml          preflight | install | config
  tasks/verify.yml
  templates/ipa-crl-vhost.conf.j2
```

Tags: `preflight`, `install`, `config`, `configure`, `verify`.

## What the vhost does

`Alias /ipa/crl/ → /var/www/crl/`. That is the whole mechanism — a document
mapping, not a redirect. A GET for
`http://ipa-ca.example.com/ipa/crl/MasterCRL.bin` is answered off disk.

`DocumentRoot` points at an **empty** directory on purpose, so only the alias
path is reachable through this vhost.

## Decisions baked in

**Port 80, no `Listen` directive.** The CDP URI in every issued certificate
carries no port, which means 80, and that string is immutable. Debian's
`ports.conf` already contains `Listen 80`, so adding another would fail with
`Address already in use`. A `Listen` line is emitted only for a non-standard
port.

**No mod_ssl.** A CDP served over TLS is a bootstrap loop: a client that cannot
yet validate a chain cannot fetch the CRL it needs to validate that chain.
Integrity comes from the CA signature, which `crl_fetch` verifies on every
consumer after download.

**No OCSP.** The DNS override captures OCSP traffic for this hostname too, so
those requests land here and get 404. Chromium is configured with
`RequireOnlineRevocationChecksForLocalAnchors: false`, so it soft-fails and
carries on. Revisit only if a client turns up that hard-fails on OCSP.

**`zz-` config prefix.** Debian loads `sites-enabled` alphabetically and the
first vhost on a port becomes the catch-all for unmatched Host headers. The
prefix stops this displacing an existing default site.

**`Cache-Control: no-store`.** Over plain HTTP any transparent proxy is a
candidate for serving a superseded CRL that no longer lists a freshly revoked
certificate.

## The access log is your distribution inventory

Every fleet pull appears here, recording what actually **arrived** rather than
what this host believed it sent. With `nextUpdate` set to a year, a host that
silently stops fetching produces no error anywhere — this log is the only place
it shows up.

```bash
# hosts that fetched in the last day
awk '{print $1}' /var/log/apache2/ipa-crl-access.log | sort -u

# and which expected hosts are missing
comm -23 <(sort expected-hosts.txt) \
         <(awk '{print $1}' /var/log/apache2/ipa-crl-access.log | sort -u)
```

Worth wiring into monitoring on day one rather than as a follow-up. Say the
word and I can make it a deployed check rather than a manual query.

## Required contract

Three values must agree with `crl_fetch` on this host:

| apache_crl | must equal | crl_fetch |
|---|---|---|
| `apache_crl_docroot` | `/var/www/crl` | `crl_fetch_publish_dir` |
| `apache_crl_filename` | `MasterCRL.bin` | `crl_fetch_filename` |
| `apache_crl_alias_path` | `/ipa/crl/` | the CDP path in your certificates |

Preflight fails if the docroot is missing, which is the usual sign `crl_fetch`
has not run yet.

## Deploy

```yaml
- name: Serve the CRL from the relay
  hosts: crl_relay
  become: true
  roles:
    - role: crl_fetch
    - role: apache_crl
```

```bash
ansible-playbook -i inventory.ini crl-relay.yml
```

Dry run: `--check --diff --skip-tags verify`. Verify performs real HTTP fetches.

## What verify proves

- `apache2ctl -S` output, so you can see the CRL vhost did not become the default
- a GET with `Host: ipa-ca.example.com` returns 200 — **before DNS is cut over**
- a GET with `Host: <each ServerAlias>` returns 200 — the fleet's path
- the served bytes actually parse as a CRL, not just that something returned 200
- directory listing is refused

## Cutover order

1. `crl_fetch` on server1 — publishes to `/var/www/crl`
2. `apache_crl` on server1 — serves it
3. `crl_fetch` on the fleet — pulls from server1's own hostname
4. **DNS last** — point `ipa-ca.example.com` at server1 in BIND

Steps 1–3 are invisible to every existing client. Only step 4 changes what
certificates resolve to, and with a short TTL it is a five-minute rollback.
