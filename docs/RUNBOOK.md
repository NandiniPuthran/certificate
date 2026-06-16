# PKI Lifecycle Runbook

## Responding to Certificate Expiry Alert

1. Check expiry status across all targets:
   ```bash
   ansible-playbook playbooks/validate_all.yml
   ```

2. Renew expiring certs:
   ```bash
   ansible-playbook playbooks/renew.yml -e renewal_threshold_days=30
   ```

3. Verify:
   ```bash
   ansible-playbook playbooks/validate_all.yml
   ```

---

## Responding to Compromised Private Key

**Time-sensitive. Execute in order.**

1. Identify the host and get serial:
   ```bash
   ansible cert_targets -m command \
     -a "openssl x509 -in /etc/ssl/company/certs/{{ cert_cn }}.crt -noout -serial" \
     --limit device01.station01.company.example
   ```

2. Revoke with keyCompromise reason:
   ```bash
   ansible-playbook playbooks/revoke.yml \
     -e target_host=device01.station01.company.example \
     -e revocation_reason=1
   ```

3. Immediately refresh CRL:
   ```bash
   ansible-playbook playbooks/setup_crl_distribution.yml --tags crl_refresh
   ```

4. Issue a new certificate:
   ```bash
   ansible-playbook playbooks/renew.yml \
     -e target_host=device01.station01.company.example \
     -e renewal_force=true
   ```

5. Update CRL cache on all clients:
   ```bash
   ansible cert_targets -m command -a "/usr/local/bin/crl-download.sh"
   ```

---

## Decommissioning a Station

1. Revoke all station certs:
   ```bash
   ansible-playbook playbooks/revoke.yml \
     -e target_station=station01 \
     -e revocation_reason=5
   ```

2. Remove from inventory.

3. Refresh CRL:
   ```bash
   ansible-playbook playbooks/setup_crl_distribution.yml --tags crl_refresh
   ```

---

## CRL Server Down / Stale CRL

Symptoms: clients log "CRL not available" or "CRL has expired"

1. Check Apache on IPA client host:
   ```bash
   ansible ipa_clients -m command -a "systemctl status apache2"
   ```

2. Check CRL age:
   ```bash
   ansible ipa_clients -m command \
     -a "openssl crl -in /var/www/crl/crl.pem -noout -nextupdate"
   ```

3. Force CRL refresh:
   ```bash
   ansible ipa_clients -m command -a "/usr/local/bin/crl-refresh.sh"
   ```

4. Check systemd timer:
   ```bash
   ansible ipa_clients -m command -a "systemctl status crl-refresh.timer"
   ```

---

## IPA CA Service Recovery

If Dogtag CA is down:

1. Restart on IPA server:
   ```bash
   ipactl restart
   # or specifically:
   systemctl restart pki-tomcatd@pki-tomcat
   ```

2. Verify:
   ```bash
   ipactl status
   ipa cert-find --sizelimit=1  # should return a result
   ```

3. Re-export profile if needed:
   ```bash
   ipa certprofile-show caIPAserviceCert --out /tmp/profile_verify.cfg
   grep "crlDistributionPoints\|authInfoAccess" /tmp/profile_verify.cfg
   ```

---

## Adding a New Station

1. Add to `inventories/production/hosts.yml`:
   ```yaml
   station03:
     hosts:
       device01.station03.company.example:
         ansible_host: 192.168.20.31
         cert_cn: device01.station03.company.example
         cert_san_dns:
           - device01.station03.company.example
         station: station03
         device_id: device01
   ```

2. Add `station03` to the `cert_targets` children.

3. Configure client revocation checking:
   ```bash
   ansible-playbook playbooks/configure_client_revocation.yml \
     -e target_station=station03
   ```

4. Issue certificates:
   ```bash
   ansible-playbook playbooks/renew.yml \
     -e target_station=station03 \
     -e renewal_force=true
   ```
