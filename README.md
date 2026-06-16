# IPA Certificate Lifecycle Management

A production-ready, modular Ansible framework for complete PKI certificate lifecycle
management using FreeIPA as CA, for **non-FreeIPA client** target hosts.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Ansible Controller                           │
│  Generates key + CSR → delegates signing → deploys certificate      │
└────────────────────────┬────────────────────────────────────────────┘
                         │
         ┌───────────────▼───────────────┐
         │     IPA Client Host           │
         │  (ipa-client01.company.example│
         │                               │
         │  • Signs CSRs via ipa CLI     │
         │  • Serves CRL via Apache      │
         │    http://crl.company.example │
         └───────────────┬───────────────┘
                         │
    ┌────────────────────┼────────────────────┐
    │                    │                    │
    ▼                    ▼                    ▼
station01/device01  station01/device02  station01/device03
(non-IPA targets)   (non-IPA targets)   (non-IPA targets)
• cert installed    • cert installed    • cert installed
• CRL cached        • CRL cached        • CRL cached
• OCSP enabled      • OCSP enabled      • OCSP enabled
```

---

## Repository Structure

```
ipa-cert-lifecycle/
├── ansible.cfg
├── requirements.yml
├── inventories/
│   └── production/
│       ├── hosts.yml                     # Inventory with station/device structure
│       └── group_vars/
│           ├── all.yml                   # Global vars (PKI endpoints, paths)
│           └── vault.yml                 # Encrypted secrets (ipa_admin_password)
├── playbooks/
│   ├── renew.yml                         # Certificate renewal
│   ├── revoke.yml                        # Certificate revocation
│   ├── setup_crl_distribution.yml        # CRL Apache server setup
│   ├── update_ipa_profile.yml            # Add CDP/OCSP to IPA cert profile
│   ├── configure_client_revocation.yml   # Client-side CRL/OCSP config
│   ├── dns_preparation.yml               # Create crl.company.example DNS record
│   └── validate_all.yml                  # End-to-end validation
├── roles/
│   ├── certificate_renewal/              # Threshold-based and forced renewal
│   ├── certificate_revocation/           # Serial-based revocation + audit trail
│   ├── crl_distribution/                 # Apache CRL server + systemd timer
│   ├── ipa_profile_update/               # Inject CDP + OCSP into Dogtag profile
│   └── client_revocation/                # CRL/OCSP config on Debian targets
└── jenkins/
    ├── Jenkinsfile.renewal               # Daily renewal pipeline
    ├── Jenkinsfile.revocation            # Manual revocation pipeline
    └── Jenkinsfile.crl_publication       # 4-hour CRL refresh pipeline
```

---

## Quick Start

### 1. Install requirements

```bash
pip install ansible
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure vault

```bash
cp inventories/production/group_vars/vault.yml.example \
   inventories/production/group_vars/vault.yml

# Edit vault.yml and set vault_ipa_admin_password
ansible-vault encrypt inventories/production/group_vars/vault.yml
echo "vault_password_here" > .vault_pass
chmod 600 .vault_pass
```

### 3. Deploy in order (first-time setup)

```bash
# Step 1: Create DNS entry for CRL FQDN
ansible-playbook playbooks/dns_preparation.yml --ask-vault-pass

# Step 2: Stand up Apache CRL distribution server on IPA client host
ansible-playbook playbooks/setup_crl_distribution.yml --ask-vault-pass

# Step 3: Update IPA certificate profile to embed CDP + OCSP URIs
ansible-playbook playbooks/update_ipa_profile.yml --ask-vault-pass

# Step 4: Configure revocation checking on all target hosts
ansible-playbook playbooks/configure_client_revocation.yml --ask-vault-pass

# Step 5: Issue/renew certificates for all targets
ansible-playbook playbooks/renew.yml --ask-vault-pass
```

---

## Certificate Renewal

### Automatic (threshold-based)

```bash
# Renew any cert expiring within 30 days (default)
ansible-playbook playbooks/renew.yml

# Custom threshold (60 days)
ansible-playbook playbooks/renew.yml -e renewal_threshold_days=60
```

### Forced renewal

```bash
# Single device
ansible-playbook playbooks/renew.yml \
  -e target_host=device01.station01.company.example \
  -e renewal_force=true

# Entire station
ansible-playbook playbooks/renew.yml \
  -e target_station=station01 \
  -e renewal_force=true
```

### Renewal workflow (per host)

```
1. Check if cert exists
2. If exists → read expiry date and serial
3. Compare days remaining vs renewal_threshold_days
4. If renewal required:
   a. Backup existing cert + key
   b. Generate new private key (community.crypto.openssl_privatekey)
   c. Generate CSR with SANs (community.crypto.openssl_csr)
   d. Copy CSR to IPA client host staging dir
   e. Submit via `ipa cert-request` on IPA client host
   f. Slurp signed cert back, push to target host
   g. Deploy CA bundle
   h. Cleanup staging files
   i. Validate: modulus match, new serial, CN check, expiry check
   j. Notify handlers (Docker, Fluent Bit, etc.)
```

---

## Certificate Revocation

### By device (serial auto-discovered from host)

```bash
ansible-playbook playbooks/revoke.yml \
  -e target_host=device01.station01.company.example \
  -e revocation_reason=5
```

### By station (all devices)

```bash
ansible-playbook playbooks/revoke.yml \
  -e target_station=station01 \
  -e revocation_reason=1
```

### By explicit serial number

```bash
ansible-playbook playbooks/revoke.yml \
  -e target_host=device01.station01.company.example \
  -e cert_serial=1A2B3C4D5E
```

### Revocation reason codes (RFC 5280)

| Code | Name                    | Use case                              |
|------|-------------------------|---------------------------------------|
| 0    | unspecified             | Generic / no specific reason          |
| 1    | keyCompromise           | Private key believed to be compromised|
| 2    | cACompromise            | CA key compromised                    |
| 3    | affiliationChanged      | Subject changed org/dept              |
| 4    | superseded              | Replaced by a new certificate         |
| 5    | cessationOfOperation    | Device decommissioned                 |
| 6    | certificateHold         | Temporary hold (reversible)           |

### Rollback considerations

Revocation is **irreversible** for reasons 0–5 and 9–10. Mitigation:

- **Before revoking**: record the serial in the audit registry (`/var/lib/ansible/cert_serials.json`) — this happens automatically
- **certificateHold** (reason 6): can be reversed with `ipa cert-revoke --revocation-reason=removeFromCRL`
- **If revoked in error**: re-issue (renew) the certificate immediately; the old serial remains revoked
- **Serial registry** at `/var/lib/ansible/cert_serials.json` provides full audit trail with host, station, reason, timestamp, and operator

---

## CRL Distribution

The IPA client host (`ipa-client01.company.example`) serves as the CRL Distribution Point.

- **URL**: `http://crl.company.example/crl.pem`
- **DER format**: `http://crl.company.example/crl.der`
- **Refresh**: systemd timer every 4 hours (matches IPA CRL generation)
- **Atomic publish**: temp file → validated → `mv` to document root

### Manual CRL refresh

```bash
# On IPA client host
sudo /usr/local/bin/crl-refresh.sh

# Or via Ansible
ansible-playbook playbooks/setup_crl_distribution.yml \
  --tags crl_refresh
```

---

## FreeIPA Profile Update

The `ipa_profile_update` role modifies the `caIPAserviceCert` Dogtag profile to
embed CRL Distribution Point and OCSP Authority Information Access extensions in
all newly-issued certificates.

**What it changes in the profile config:**

```properties
# CRL Distribution Point (OID 2.5.29.31)
policyset.serverCertSet.12.default.class_id=crlDistributionPointsExtDefaultImpl
policyset.serverCertSet.12.default.params.crlDistributionPointsPointName_0=http://crl.company.example/crl.pem

# OCSP AIA (OID 1.3.6.1.5.5.7.1.1)
policyset.serverCertSet.13.default.class_id=authInfoAccessExtDefaultImpl
policyset.serverCertSet.13.default.params.authInfoAccessADLocation_0=http://ocsp.company.example
```

**Important**: After the profile update, renew all existing certificates so they
contain the new extensions. Existing certs issued under the old profile will not
have CDP/OCSP embedded until renewed.

---

## Client Revocation Checking

### CRL checking

Each target host:
1. Downloads CRL to `/etc/ssl/company/crl/crl.pem` every 60 minutes (systemd timer)
2. Validates CRL is valid PEM and not expired before replacing local cache
3. OpenSSL configuration references local CRL for revocation checking

### OCSP checking

FreeIPA's built-in Dogtag OCSP responder is exposed at the IPA server. The
`ocsp_uri` variable points clients to it. OCSP stapling at the server side is
the preferred pattern for production.

### Limitations

| Application  | CRL Support | OCSP Support | Notes                                    |
|-------------|------------|-------------|------------------------------------------|
| OpenSSL CLI  | ✓ Full     | ✓ Full      | Via openssl.cnf + CRLFile                |
| Docker       | Partial    | Via system  | Trusts system CA; CRL via system OpenSSL |
| Fluent Bit   | Indirect   | Via system  | Uses system OpenSSL; no explicit CRL API |
| Custom apps  | App-level  | App-level   | Must use SSL_CTX_set_verify() explicitly |

---

## Station / Device Inventory Model

```yaml
# hosts.yml pattern
station01:
  hosts:
    device01.station01.company.example:
      cert_cn: device01.station01.company.example
      station: station01
      device_id: device01
    device02.station01.company.example:
      cert_cn: device02.station01.company.example
      station: station01
      device_id: device02
```

Target selection at playbook runtime:

```bash
# All stations
ansible-playbook playbooks/renew.yml

# One station
ansible-playbook playbooks/renew.yml -e target_station=station01

# One device
ansible-playbook playbooks/renew.yml -e target_host=device01.station01.company.example
```

---

## Security Considerations

### Private key handling
- Keys generated **on the target host** — private key never leaves the host
- Key directory mode: `0710` (root:root)
- Key file mode: `0600`
- Backups include keys — ensure backup directory is restricted (`0710`)

### Signing host staging
- CSR/signed cert staging in `/tmp/ansible_cert_signing` (mode `0700`)
- Staging files deleted immediately after use
- Use a dedicated service account for `kinit`; prefer keytab over password

### Credentials
- IPA admin password stored only in Ansible Vault
- Never passes password on command line — use kinit with keytab where possible
- Jenkins credentials stored in Jenkins Credential Store, not in code

### CRL Distribution
- Served over HTTP only (standard for CRL per RFC 5280 — HTTPS creates chicken-and-egg problem)
- Apache directory listing disabled
- Only `crl.pem` and `crl.der` accessible via FilesMatch restriction

### Audit trail
- Every revocation written to JSON registry with operator, timestamp, reason, host, station
- Registry: `/var/lib/ansible/cert_serials.json`

---

## Validation Checklist

```bash
# Run full validation suite
ansible-playbook playbooks/validate_all.yml

# Individual checks:

# Is the CRL reachable?
curl -v http://crl.company.example/crl.pem | openssl crl -noout -text

# Check a cert's embedded CDP/OCSP
openssl x509 -in /etc/ssl/company/certs/<host>.crt -noout -text \
  | grep -A2 "CRL Distribution\|OCSP"

# Verify a cert against CRL
openssl verify \
  -CAfile /etc/ssl/company/ca/ipa-ca.crt \
  -CRLfile /etc/ssl/company/crl/crl.pem \
  -crl_check \
  /etc/ssl/company/certs/<host>.crt

# Check OCSP status for a cert
openssl ocsp \
  -issuer /etc/ssl/company/ca/ipa-ca.crt \
  -cert /etc/ssl/company/certs/<host>.crt \
  -url http://ocsp.company.example \
  -CAfile /etc/ssl/company/ca/ipa-ca.crt \
  -text
```

---

## Rollback Strategy

| Scenario | Rollback Action |
|----------|-----------------|
| Bad cert deployed | Restore from `cert_backup_dir` (backup created before each renewal) |
| IPA profile update broke issuance | `ipa certprofile-mod --file <backup>.cfg` from `profile_backup_dir` |
| CRL server down | Target hosts retain local CRL cache; grace period per app config |
| Revoked cert in error | Immediately renew; old serial stays revoked; new serial is valid |
| Dogtag restart failed | `ipactl restart`; check `/var/log/pki/pki-tomcat/ca/debug` |

---

## Jenkins Pipelines

| Pipeline | File | Trigger | Purpose |
|----------|------|---------|---------|
| Renewal | `Jenkinsfile.renewal` | Daily 06:00 + manual | Renew expiring certs |
| Revocation | `Jenkinsfile.revocation` | Manual only | Revoke with confirmation gate |
| CRL Publication | `Jenkinsfile.crl_publication` | Every 4h | Keep CRL distribution current |

---

## DNS Assumptions

- DNS is FreeIPA-integrated BIND
- `crl.company.example` A record created via `ipa dnsrecord-add`
- If using external DNS (Route53, Infoblox), replace the IPA DNS tasks in `playbooks/dns_preparation.yml` with the appropriate Ansible module
- CNAME records are **not recommended** for CRL/OCSP endpoints (some clients don't follow CNAME in CDP)
