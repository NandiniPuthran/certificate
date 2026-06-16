# Implementation Notes & Design Rationale

## Why the IPA Client Host is the CRL Distribution Point

FreeIPA generates CRLs on the CA (Dogtag PKI, running inside the IPA server).
The CRL file is available locally on IPA-enrolled hosts at `/etc/ipa/ca.crl`,
which is refreshed by the `certmonger` and IPA CRL process every 4 hours.

Rather than exposing the IPA server directly to CRL consumers (which may be in
different network segments), we:

1. Use an existing IPA client host as a CRL relay / distribution point
2. Serve the CRL via a minimal Apache virtual host over HTTP

**Why HTTP and not HTTPS for CRL?**
RFC 5280 and practical PKI deployment intentionally use HTTP for CRL Distribution
Points. If CRL was served over HTTPS, a client that needed to validate the TLS
certificate serving the CRL would face a circular dependency. HTTP for CRL is
the industry standard — all major CAs (DigiCert, Let's Encrypt, etc.) use HTTP
CDPs.

---

## Why We Use `ipa cert-request` via Delegation, Not `certmonger`

`certmonger` is designed for IPA-enrolled hosts. Our target hosts are
intentionally **not** IPA-enrolled (they may be in restricted network zones,
air-gapped environments, or managed by separate teams). Therefore:

- The **Ansible controller** generates the private key and CSR on the target host
- The CSR is **slurped** (base64 via Ansible) to avoid writing to the controller
- The CSR is pushed to an IPA client host (the signing relay)
- `ipa cert-request` is run on the signing relay with the correct Kerberos TGT
- The signed certificate is slurped back and pushed to the target host

This preserves the private key on the target and avoids enrolling the target
into IPA.

---

## Why `community.crypto` Instead of `openssl` CLI for Key/CSR Generation

`community.crypto.openssl_privatekey` and `openssl_csr` are:

1. **Idempotent** — they check if the existing key/CSR still satisfies the
   requested parameters before regenerating. The `force` flag overrides this.
2. **Declarative** — you describe the desired state (key size, type, SANs)
   rather than scripting `openssl genrsa` and managing return codes.
3. **Ansible-native** — return values integrate cleanly with `changed_when`
   and handlers.

---

## FreeIPA Profile Modification: Policy Set Numbering

Dogtag certificate profiles are Java property files. Each certificate extension
is a numbered entry in a `policyset`. The profile template:

1. Exports the **existing** profile via `ipa certprofile-show --out`
2. The Jinja2 template **appends** entries 12 and 13 to the `serverCertSet`
3. Updates the `policyset.serverCertSet.list` property to include 12 and 13

**Risk**: If the existing profile already uses slots 12 or 13, there will be a
conflict. Mitigation: the exported profile is backed up, and the role checks for
the URI in the re-imported profile as validation. For environments where the
default profile has been heavily customised, inspect the export and adjust
`_cert_profile_start_index` (a future extension point).

---

## Serial Number Handling: Hex vs Decimal

FreeIPA's `ipa cert-show` and `ipa cert-revoke` accept **decimal** serial
numbers, while OpenSSL reports serials in **hex**. The revocation role converts
with Jinja2:

```jinja2
_serial_decimal: "{{ _serial_item | int(base=16) }}"
```

This is why the serial registry stores both `serial_hex` and `serial_decimal`.

---

## CRL Refresh: Atomic File Replace

The CRL refresh script uses:
```bash
TMP=$(mktemp -p /var/www/crl crl.XXXXXX.pem)
cp /etc/ipa/ca.crl "$TMP"
# ... validate ...
mv "$TMP" /var/www/crl/crl.pem
```

`mv` on the same filesystem is atomic at the kernel level. This prevents Apache
from serving a partially-written CRL file during the refresh.

---

## Fluent Bit Revocation Limitation

Fluent Bit (as of version 3.x) does not expose a configuration option to specify
a CRL file for TLS validation. It uses OpenSSL internally but does not pass
`SSL_CTX_load_verify_locations` with a CRL parameter.

**Practical mitigations in order of preference:**
1. Configure OCSP stapling on TLS servers that Fluent Bit connects to
2. Ensure `openssl.cnf` on the host configures system-level CRL checking
   (affects OpenSSL library calls from all processes, including Fluent Bit)
3. Use a TLS-terminating proxy (nginx, HAProxy) in front of the Fluent Bit
   destination, with proper revocation checking at the proxy layer

---

## Docker CA Trust vs. Revocation

Docker daemon's TLS (for registry communication) trusts the system CA store.
After deploying `update-ca-certificates`, Docker must be restarted to pick up
the new trust anchors.

Docker does **not** perform CRL or OCSP checking natively. Revocation checking
for Docker registry connections requires a TLS-inspecting proxy or a registry
configured with OCSP stapling.

For container workloads that need to verify certificates at runtime (e.g., a
Go/Python service in a container), mount the CA bundle and CRL into the container
and use application-level certificate verification.

---

## Kerberos Authentication on IPA Client Host

The role uses `kinit -kt /etc/krb5.keytab` which authenticates as the **host
principal** of the IPA client host. This principal must have permission to
request certificates via the specified profile.

For production, consider:
1. Creating a dedicated service principal (e.g., `ansible-cert-svc/ipa-client01`)
2. Granting it cert-request privilege via IPA RBAC
3. Using that principal's keytab instead of the host keytab

This avoids using the host's Kerberos identity for application-level PKI
operations, improving auditability.

---

## Idempotency Design

Every task is designed to be safely re-runnable:

| Component | Idempotency mechanism |
|-----------|----------------------|
| Private key generation | `community.crypto.openssl_privatekey` — skips if params unchanged |
| CSR generation | `community.crypto.openssl_csr` — skips if params unchanged |
| Apache vhost | Template + `changed_when` only if file content changes |
| Systemd timer | `systemd` module with `state: started enabled: true` — no-op if already running |
| DNS record | `ipa dnsrecord-add` — checks existence first, treats "already exists" as success |
| CA trust | `copy` module — skips if file content unchanged |
| CRL refresh | Script validates and uses atomic `mv` — safe to re-run |
| IPA profile | Exports current profile, appends extensions, checks for URI presence before importing |
| Serial registry | Read → merge → write JSON — safe concurrent reads; last-write-wins for concurrent runs |

---

## Scaling Considerations

- **`serial` in playbooks**: `renew.yml` defaults to `10%` batch size to avoid
  overloading the IPA CA with simultaneous cert-request calls
- **Forks**: `ansible.cfg` sets `forks=10`; increase for larger inventories
- **Staging cleanup**: per-host namespaced staging files
  (`/tmp/ansible_cert_signing/device01.station01.company.example.csr`) prevent
  collision when multiple hosts are processed concurrently
- **CRL size**: for large environments, consider `crl_der_filename` as default
  (DER is more compact); modern clients handle PEM fine
