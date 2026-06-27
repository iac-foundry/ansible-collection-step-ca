# blueprints.step_ca

An Ansible collection implementing a single-tier Smallstep `step-ca` PKI as a building block for
internal TLS issuance in the Blueprints framework.

**Status:** Stage 7 implementation scaffold (structure ready, tasks pending completion)

## Overview

`blueprints.step_ca` stands up a Smallstep `step-ca` instance as an internal certificate authority
(single-tier, root-only) and provides an interface for any component to request and consume
TLS certificates on demand.

- **No cross-collection dependencies** — this collection is org-neutral and independent, converging
  on a clean host given only its own caller-supplied variables (Design Principles §5).
- **Two roles:**
  - `container_server`: Initialize and run step-ca as a container, exposing the HTTPS API
  - `issue_certificate`: Request a leaf certificate from a running step-ca instance
- **Design:** See [BLUEPRINTS_STEP_CA_PKI_DESIGN.md](../../docs/design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md)
  and the single-tier rationale in [LADR-004](../../docs/decisions/LADR-004-step-ca-single-tier-pki.md).

## Quick start

### Bring up a step-ca root CA as a container

```yaml
- name: Converge step-ca
  hosts: sec01  # or any container-capable host
  tasks:
    - name: Start step-ca container
      include_role:
        name: blueprints.step_ca.container_server
      vars:
        step_ca_server_image: 'smallstep/step-ca:0.27.0'  # pinned tag (caller chooses)
        step_ca_server_provisioner_password: "{{ vault_provisioner_password }}"  # from secret store
        step_ca_server_https_port: 443
        step_ca_server_data_dir: "/var/lib/step-ca"

    - name: Save root cert for distribution
      copy:
        content: "{{ step_ca_server_root_ca_cert }}"
        dest: /etc/ssl/certs/step-ca-root.pem
```

### Issue a certificate

```yaml
- name: Issue certificate for Vault
  hosts: sec01
  tasks:
    - name: Request Vault TLS certificate
      include_role:
        name: blueprints.step_ca.issue_certificate
      vars:
        step_ca_issue_ca_url: "https://step-ca.local:443"
        step_ca_issue_ca_fingerprint: "{{ step_ca_server_ca_fingerprint }}"  # from previous converge
        step_ca_issue_provisioner_password: "{{ vault_provisioner_password }}"
        step_ca_issue_certificate_common_name: "vault.local"
        step_ca_issue_certificate_subject_alt_names:
          - vault.local
          - vault.example.com
        step_ca_issue_certificate_validity_days: 30

    - name: Pass cert to Vault
      include_role:
        name: blueprints.vault.container_server
      vars:
        vault_server_image: 'vault:latest'
        vault_server_tls_cert: "{{ step_ca_issue_certificate_pem }}"
        vault_server_tls_key: "{{ step_ca_issue_certificate_key_pem }}"
```

## Roles

### container_server

Stand up a step-ca root CA in a container.

- **Inputs:** `step_ca_server_image` (required), `step_ca_server_provisioner_password` (required), and
  optional config (port, data directory, service user, CA cert identity).
- **Outputs:** `step_ca_server_root_ca_cert` (PEM), `step_ca_server_ca_fingerprint` (public).
- **See:** [roles/container_server/README.md](roles/container_server/README.md)

### issue_certificate

Request a leaf certificate from a running step-ca instance.

- **Inputs:** `step_ca_issue_ca_url` (required), `step_ca_issue_ca_fingerprint` (required),
  `step_ca_issue_provisioner_password` (required), `step_ca_issue_certificate_common_name` (required),
  and optional SANs, validity period, and output paths.
- **Outputs:** `step_ca_issue_certificate_pem`, `step_ca_issue_certificate_key_pem` (both no_log),
  `step_ca_issue_certificate_chain_pem`, `step_ca_issue_certificate_not_after`,
  `step_ca_issue_certificate_days_remaining`.
- **See:** [roles/issue_certificate/README.md](roles/issue_certificate/README.md)

## Architecture

```
Platform layer (caller)
  ├─ include_role: container_server (setup step-ca once)
  │  └─ step_ca_server_root_ca_cert → distributed to clients
  │
  └─ include_role: issue_certificate (on-demand cert requests)
     ├─ HTTPS to container_server's API
     └─ returns cert + key + expiry facts for wiring into consumers
```

See the [design document](../../docs/design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md) for topology,
requirements, trade-offs, and risks.

## Key design principles

1. **Single-tier (root-only) PKI:** No intermediate CA layering in v1. This is an explicit,
   documented trade-off in [LADR-004](../../docs/decisions/LADR-004-step-ca-single-tier-pki.md),
   appropriate to small-environment scale.

2. **No cross-collection dependencies:** `galaxy.yml` → `dependencies: {}`. Composition is a
   platform-layer concern, not a collection concern (Design Principles §1, §5).

3. **All inputs are caller-supplied variables:** No hardcoded hostnames, no secret retrieval,
   no external discovery (Principle 2). Variables follow the `step_ca_<role>_<thing>` prefix
   (Variable Standards §1).

4. **Idempotence-first:** Second converge reports zero changes. Container tasks use
   `community.docker.docker_container` in declarative mode (Principle 4).

5. **Fail-fast on missing secrets:** No placeholder defaults for passwords or other required
   secrets. Missing inputs fail the play immediately with a clear error message (Security §3).

6. **No silent global changes:** Trust-store installation is explicitly deferred (Risk R-1).
   Root cert *retrieval* is solved natively by step-ca's own CA API (`/roots.pem` endpoint).
   Firewall configuration is a platform responsibility, not collection code (Decision 9).

7. **Short-lived defaults force deliberate renewal:** Default leaf validity is 30 days (Decision 3),
   pushing renewal responsibility to the platform layer rather than silently deferring it.

## Non-goals

- Does **not** install the root CA into any host's system-wide trust store (`/etc/ssl/certs`, etc.).
  That is a host-wide blast-radius change requiring platform-layer governance. Root cert *retrieval*
  is already solved natively by step-ca.
- Does **not** implement automated renewal. Renewal is a platform/consumer concern (see the
  monitoring/renewal recipes in the role READMEs).
- Does **not** configure revocation (CRL/OCSP). Short-lived certs (30-day default) make revocation
  less critical in small environments; it is deferred to v2+ if needed.
- Does **not** deploy or wire any other `blueprints.*` component. Composition is the caller's responsibility.
- Does **not** configure multi-instance clustering or HA. Single-instance is the v1 model (see LADR-004).

## Testing

Each role includes a `molecule/default` scenario:

```bash
cd roles/container_server && molecule test -s default
cd roles/issue_certificate && molecule test -s default
```

CI/CD (GitHub Actions) validates:
- No hidden cross-collection dependencies (`ci/no_hidden_deps_guard.sh`)
- Ansible lint (production profile)
- Molecule test for each role

## Requirements

- Ansible 2.16+
- `community.docker` collection (for container lifecycle management)
- Docker or Docker-API-compatible runtime on the target host (pre-installed; not provisioned by
  this collection)
- Network reachability between the caller and the step-ca API endpoint

## Related documentation

- **Design:** [BLUEPRINTS_STEP_CA_PKI_DESIGN.md](../../docs/design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md)
- **LADR-004:** [Single-tier PKI decision](../../docs/decisions/LADR-004-step-ca-single-tier-pki.md)
- **Variable standards:** [BLUEPRINTS_VARIABLE_STANDARDS.md](../../docs/standards/BLUEPRINTS_VARIABLE_STANDARDS.md)
- **Role layout:** [BLUEPRINTS_ROLE_LAYOUT.md](../../docs/standards/BLUEPRINTS_ROLE_LAYOUT.md)
- **Secret consumption:** [BLUEPRINTS_SECRET_CONSUMPTION.md](../../docs/standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
- **Design principles:** [BLUEPRINTS_DESIGN_PRINCIPLES.md](../../docs/design/BLUEPRINTS_DESIGN_PRINCIPLES.md)

## License

MIT
