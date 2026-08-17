# crl_fetch

Keeps a validated, current copy of the FreeIPA CRL on local disk.

The same role runs on **both** sides of the pull model — only `group_vars`
differ:

| Host group | Fetches from | Publishes to |
|---|---|---|
| `crl_relay` (server1) | the FreeIPA CA over the WAN | `/var/www/crl` |
| `certificate` (fleet) | server1 over the LAN | `/etc/ssl/crl` |

The role itself never distinguishes the two. It only ever references
`{{ crl_fetch_publish_dir }}` and `{{ crl_fetch_source_url }}`; Ansible resolves
those per host from `group_vars/<groupname>.yml`.

## Layout

```
crl_fetch/
  defaults/main.yml            22 variables
  handlers/main.yml
  meta/main.yml
  tasks/main.yml               → configure, then verify
  tasks/configure.yml          preflight | layout | scripts | timer
  tasks/verify.yml
  templates/crl-fetch.sh.j2         the fetch script
            crl-fetch.service.j2    what to run
            crl-fetch.timer.j2      when to run it
            crl-reload.sh.j2        tell services the CRL changed
```

Tags still work at phase granularity: `preflight`, `layout`, `scripts`,
`timer`, `configure`, `verify`.

## What a run does

1. **Download** with `--fail` and `--max-redirs 0`, so an error page or a
   redirect never becomes the CRL.
2. **Checksum.** Identical to what is published → exit immediately. The CA
   regenerates every 4h and this runs every 3h, so most runs stop here without
   parsing or verifying anything.
3. **Verify the CA signature.** The transport is plain HTTP, so this is the
   only thing distinguishing a genuine CRL from a forged empty one dated today
   — which would revoke nothing and pass both the checksum and timestamp
   checks.
4. **Compare `lastUpdate`.** Older than or equal to what is published → refuse.
   `lastUpdate` rather than `nextUpdate` because `nextUpdate` is derived from
   the CA's grace period, so changing `nextUpdateGracePeriod` would shift the
   ordering underneath the comparison.
5. **Publish atomically** — write beside the target, then rename, so no reader
   ever sees a half-written CRL.
6. **Reload services** via `ExecStartPost=+/usr/local/sbin/crl-reload`, if
   `crl_fetch_reload_services` is populated.

Any failure leaves the previously published CRL untouched. Stale-but-genuine
always beats unproven.

## crl-reload

Services read the CRL once at startup and hold it in memory. Without this, the
fetch succeeds, the file is current on disk, and a certificate revoked this
morning is still accepted — everything reports success while nothing changed.

The `+` prefix is load-bearing: `crl-fetch` runs unprivileged with
`NoNewPrivileges=true`, which blocks `sudo` outright. `+` makes systemd run
that one command as root, outside the sandbox, so parsing untrusted bytes stays
unprivileged and only the reload gets privilege.

It compares the CRL checksum against a stored marker, so it is a no-op on the
majority of runs where nothing changed. The marker only advances when **every**
reload succeeded — otherwise a single failed reload would leave that service on
an old CRL forever, because the checksum would already match next time.

## Required group_vars

```yaml
# group_vars/crl_relay.yml
crl_fetch_ipa_server_fqdn: ipa1.sitea.example.com   # the CA's OWN name
crl_fetch_ca_cert: /etc/ssl/certs/ipa-ca.crt
crl_fetch_publish_dir: /var/www/crl
crl_fetch_interval: "*-*-* 00/3:00:00"
crl_fetch_reload_services: []                        # nothing consumes it here
```

```yaml
# group_vars/certificate.yml
crl_fetch_ipa_server_fqdn: server1.siteb.example.com
crl_fetch_source_url: "http://{{ crl_fetch_ipa_server_fqdn }}/ipa/crl/MasterCRL.bin"
crl_fetch_publish_dir: /etc/ssl/crl
crl_fetch_pem_filename: ipa-crl.pem
crl_fetch_ca_cert: /etc/ssl/certs/ipa-ca.crt
crl_fetch_interval: "*-*-* 00/3:45:00"               # offset from the relay
crl_fetch_randomized_delay: 900
crl_fetch_reload_services: []                        # POPULATE THIS
```

The filename must match the inventory group name exactly. A mismatch fails
**silently** — no warning, and hosts quietly fall back to role defaults. Verify
once per group:

```bash
ansible -i inventory.ini <a-fleet-host> -m debug -a "var=crl_fetch_publish_dir"
```

Never put server1 in the `certificate` group. Keep the two disjoint.

## Deploy order

Relay first, verified, then the fleet. Until server1 publishes there is nothing
to fetch, and fleet preflight fails on every host — correctly, since it does a
`HEAD` against the source before doing anything else.

Dry run: `--check --diff --skip-tags verify`. The verify phase performs a real
fetch and cannot run in check mode.

## Testing by hand

```bash
sudo -u crlfetch /usr/local/sbin/crl-fetch          # one run
sudo -u crlfetch /usr/local/sbin/crl-fetch          # again: expect "unchanged"

openssl crl -inform DER -in /etc/ssl/crl/MasterCRL.bin -noout -lastupdate -nextupdate
systemctl list-timers crl-fetch.timer --no-pager
journalctl -t crl-fetch -t crl-reload --since '1 day ago'
```

The signature rejection path is the one worth proving deliberately: point
`crl_fetch_ca_cert` at an unrelated CA certificate, run the script, and confirm
it refuses to publish and leaves the existing file untouched.
