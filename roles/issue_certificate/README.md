# blueprints.step_ca.issue_certificate

Request a single leaf TLS certificate from a running step-ca instance, given a caller-supplied
Common Name and Subject Alternative Names. Returns the signed cert, private key, and chain as
Ansible facts; optionally writes them to paths on the target host.

## Deployment model

`client` / on-demand issuance (can run on any host with network reachability to the CA).

## What it does

Converges on a host to:
1. Assert that the caller has supplied all required inputs (CA URL, fingerprint, provisioner password, CN)
2. Validate the CA's HTTPS endpoint is reachable using TOFU (trust-on-first-use) with the caller-supplied fingerprint
3. Request a single leaf certificate from the CA, authenticated via the provisioner password
4. Return the signed certificate, private key, and certificate chain as Ansible facts (no_log on the key)
5. Extract and return certificate expiry facts (`step_ca_issue_certificate_not_after` and `_days_remaining`)
   for platform-layer monitoring/alerting without parsing the cert file itself
6. Optionally write the cert, key, and chain to caller-supplied file paths on the target host (consumer's choice)

The certificate and chain are public artifacts (expiry is encoded in the cert itself, not new
information). The private key is secret and always marked `no_log`.

## Inputs

All configuration is supplied via caller variables. Required inputs have no default and fail fast
when missing. See `meta/argument_specs.yml` for the complete, typed interface.

### Required inputs
- `step_ca_issue_ca_url`: HTTPS URL to the step-ca CA API endpoint.
  Example: `'https://step-ca.local:443'` or `'https://sec01.example.com:443'`
- `step_ca_issue_ca_fingerprint`: SHA-256 fingerprint of the step-ca root certificate. Used for
  TOFU-based client bootstrap (public, not secret). Example: `'abcd1234...'`
- `step_ca_issue_provisioner_password`: Secret password for the JWK provisioner. Must be supplied
  by caller; fail-fast if missing. Mark as `no_log` in your playbook.
- `step_ca_issue_certificate_common_name`: Common Name (CN) for the leaf certificate.
  Example: `'vault.local'` or `'service.example.com'`

### Key optional inputs
- `step_ca_issue_certificate_subject_alt_names` (default `[]`): List of Subject Alternative Names (SANs).
  Example: `['vault.local', 'vault.example.com', '10.0.1.10']`
- `step_ca_issue_certificate_validity_days` (default `30`): Validity period in days. Default 30 per
  Design Decision 3, forcing deliberate renewal at the platform layer. Caller-overridable.
- `step_ca_issue_certificate_output_cert_path` (default `""`): If set, write the certificate to this path on the target host.
- `step_ca_issue_certificate_output_key_path` (default `""`): If set, write the private key to this path (mode 0600).
- `step_ca_issue_certificate_output_chain_path` (default `""`): If set, write the certificate chain (root only) to this path.

## Outputs (facts)

This role sets the following Ansible facts for the platform layer to consume:

### Secrets (no_log: true)
- `step_ca_issue_certificate_pem`: The issued leaf certificate in PEM format.
- `step_ca_issue_certificate_key_pem`: The issued private key in PEM format.

### Public (not secret)
- `step_ca_issue_certificate_chain_pem`: The certificate chain in PEM format (root CA only; no intermediate exists in single-tier PKI).
- `step_ca_issue_certificate_not_after`: Certificate expiry timestamp (ISO8601 format).
- `step_ca_issue_certificate_days_remaining`: Certificate validity remaining at issuance time (integer, calculated from Not After).

Example platform-layer usage:
```yaml
- name: Issue certificate for Vault
  include_role:
    name: blueprints.step_ca.issue_certificate
  vars:
    step_ca_issue_ca_url: "https://step-ca.local:443"
    step_ca_issue_ca_fingerprint: "{{ vault_ca_fingerprint }}"  # from previous container_server converge
    step_ca_issue_provisioner_password: "{{ vault_provisioner_password }}"  # from secret store
    step_ca_issue_certificate_common_name: "vault.local"
    step_ca_issue_certificate_subject_alt_names:
      - vault.local
      - vault.example.com
      - "10.0.1.10"
    step_ca_issue_certificate_validity_days: 30

- name: Alert if cert expires soon
  ansible.builtin.assert:
    that:
      - step_ca_issue_certificate_days_remaining | int > 7
    fail_msg: "Certificate expires in {{ step_ca_issue_certificate_days_remaining }} days — renewal needed"

- name: Pass issued cert to another component (e.g., Vault)
  include_role:
    name: blueprints.vault.container_server
  vars:
    vault_server_image: 'vault:latest'
    vault_server_tls_cert: "{{ step_ca_issue_certificate_pem }}"
    vault_server_tls_key: "{{ step_ca_issue_certificate_key_pem }}"
```

## Non-goals

- Does not retrieve the provisioner password or CA fingerprint — the caller resolves them and
  passes them in via variables (see
  [BLUEPRINTS_SECRET_CONSUMPTION.md](../../../../docs/standards/BLUEPRINTS_SECRET_CONSUMPTION.md)).
- Does not implement automated renewal or rotation. Renewal is a platform/consumer-layer concern.
  The default 30-day validity forces renewal to happen deliberately at the platform layer. A consumer
  that needs automated renewal can re-run this role on a schedule (e.g. via a scheduled platform
  playbook run, or a `vault-agent`-style sidecar, as the roadmap shows for Phases 3/5) rather than
  building renewal automation here.
- Does not configure any application-level TLS settings (e.g. loading the cert into a specific
  service's configuration). The cert and key are returned as facts; wiring them into a consumer is
  the platform layer's responsibility (see the Design Principles §5 on composition).
- Does not deploy or wire any other `blueprints.*` component — composition is the caller's responsibility.

## Renewal recipe (informational)

The default 30-day validity in `step_ca_issue_certificate_validity_days` is short by design,
enforcing a deliberate renewal story at the platform layer rather than silently deferring renewal
until it becomes an incident.

Example renewal strategy (platform-layer code):
```yaml
# Scheduled platform playbook (e.g., daily check)
- name: Check Vault cert expiry and renew if needed
  hosts: sec01
  tasks:
    - name: Issue certificate for Vault (renews if already converged)
      include_role:
        name: blueprints.step_ca.issue_certificate
      vars:
        step_ca_issue_ca_url: "https://step-ca.local:443"
        step_ca_issue_ca_fingerprint: "{{ vault_ca_fingerprint }}"
        step_ca_issue_provisioner_password: "{{ vault_provisioner_password }}"
        step_ca_issue_certificate_common_name: "vault.local"
        step_ca_issue_certificate_validity_days: 30
        step_ca_issue_certificate_output_cert_path: "/etc/vault/tls/vault.crt"
        step_ca_issue_certificate_output_key_path: "/etc/vault/tls/vault.key"

    - name: Reload Vault if cert was updated
      ansible.builtin.debug:
        msg: "Vault reload would happen here if cert changed (check Vault's own reload mechanism)"
```

This collection ships no alerting/monitoring code itself — the expiry facts are the entire contract.
What a platform layer does with them (alert, renew on schedule, or otherwise act) is its concern.

## Monitoring recipe (informational)

Because `issue_certificate` returns `step_ca_issue_certificate_days_remaining` as an ordinary fact,
a platform layer can wire cert-expiry alerting without parsing PEM files itself:

```yaml
- name: Converge step-ca issuance
  include_role:
    name: blueprints.step_ca.issue_certificate
  vars:
    # ... required inputs ...

- name: Alert if expiry is imminent
  ansible.builtin.assert:
    that:
      - step_ca_issue_certificate_days_remaining | int > 7
    fail_msg: "Vault cert expires in {{ step_ca_issue_certificate_days_remaining }} days"

# Or feed into your platform's own alerting system:
- name: Send expiry metric to monitoring backend
  ansible.builtin.uri:
    url: "{{ monitoring_backend_url }}/metrics"
    method: POST
    body_format: json
    body:
      metric: "certificate.days_remaining"
      service: "vault"
      value: "{{ step_ca_issue_certificate_days_remaining }}"
```

## Dependencies

- Network reachability to the step-ca CA API endpoint (supplied by caller; no discovery).
- The step-ca instance itself must already be running (via `blueprints.step_ca.container_server` or
  `blueprints.step_ca.systemd_server` when it's implemented).
- `step` CLI (Smallstep's client binary), or Ansible's `uri` module for REST API calls.
  Implementation detail left to Stage 7.
- Ansible 2.16+

## Security considerations

- **Provisioner password:** This is a caller-supplied secret. Never store it in this collection or
  pass it to untrusted consumers. The role sets `no_log: true` on any task consuming it.
- **Private key:** The issued private key is marked `no_log: true` in all fact outputs. If output
  paths are specified, the key file is written with mode `0600` (readable by owner only).
- **TOFU bootstrap:** The CA fingerprint is public (not secret), supplied by caller, and used to
  establish first-contact trust with the CA's HTTPS endpoint. TOFU is inherently trust-on-first-use;
  subsequent verifications use standard TLS certificate validation.

## Reliability & Idempotence

- **Idempotence:** Calling this role multiple times on the same host issues a new certificate each time.
  There is no built-in deduplication — if you want to issue once per day or once per week, implement
  that logic in your platform playbook (e.g. `changed_when` guards based on cert age, or a scheduled
  cron job).
- **Single issuance per invocation:** The role requests one certificate and returns. No retry-as-secret-bypass;
  if the CA is unreachable or the request fails, the failure surfaces to the caller, never silently
  defaults to an unsigned/self-signed fallback.

## See also

- [Design document](../../../../docs/design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md)
- [LADR-004: Single-tier PKI decision](../../../../docs/decisions/LADR-004-step-ca-single-tier-pki.md)
- [blueprints.step_ca.container_server](../container_server/README.md) — stand up the CA
- [BLUEPRINTS_VARIABLE_STANDARDS.md](../../../../docs/standards/BLUEPRINTS_VARIABLE_STANDARDS.md)
- [BLUEPRINTS_SECRET_CONSUMPTION.md](../../../../docs/standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
