# blueprints.step_ca.container_server

Initialize and run Smallstep step-ca as a container, exposing the root CA via HTTPS API and at
least one provisioner for leaf certificate issuance.

## Deployment model

`container_server` (server — runs the CA itself on a target host).

## What it does

Converges on a host to:
1. Assert that the caller has supplied a container image reference and provisioner secret
2. Create a dedicated service user and durable data directory with restrictive permissions (0700)
3. Render step-ca's own configuration format (`ca.json`) from caller-supplied variables
4. Generate a new root key/cert pair, or accept a caller-supplied pair (FR-1)
5. Configure exactly one default JWK (password-protected) provisioner from the provisioner password
6. Start/restart the step-ca container via `community.docker.docker_container` in declarative mode
   (idempotent on repeat converge)
7. Expose the root CA certificate (public, not secret) and CA fingerprint as Ansible facts for the
   platform layer to consume, distribute, or write to disk

The root cert and fingerprint are public artifacts (the cert is already public; the fingerprint is
used for TOFU-based client bootstrap). Neither is sensitive and requires no `no_log`.

## Inputs

All configuration is supplied via caller variables. Required inputs have no default and fail fast
when missing. See `meta/argument_specs.yml` for the complete, typed interface.

### Required inputs
- `step_ca_server_image`: Container image reference (caller-supplied; no org default).
  Example: `'smallstep/step-ca:0.27.0'`
- `step_ca_server_provisioner_password`: Secret password for the default JWK provisioner.
  Must be supplied by caller; fail-fast if missing. Mark as `no_log` in your playbook.

### Key optional inputs
- `step_ca_server_https_port` (default 443): HTTPS port for the CA API.
- `step_ca_server_data_dir` (default `/var/lib/step-ca`): Persistent data directory for root key,
  cert, and provisioner config. Must be durable across container restart.
- `step_ca_server_service_user`/`_group` (default `step-ca`): System user/group owning the data
  directory. Container runs as this user (if base image supports unprivileged mode).
- `step_ca_server_container_name` (default `step-ca`): Name of the running container.
- `step_ca_server_log_level` (default `info`): Logging level (`debug`, `info`, `warn`, `error`).
- `step_ca_server_root_ca_name`, `_cn`, `_organization`: Human-readable identifiers for the
  generated root certificate.

## Outputs (facts)

This role sets the following Ansible facts for the platform layer to consume:

- `step_ca_server_root_ca_cert`: The root CA certificate in PEM format (public; not secret).
- `step_ca_server_ca_fingerprint`: SHA-256 fingerprint of the root cert (public; used for TOFU
  client bootstrap with `step ca bootstrap --fingerprint`).

Example platform-layer usage:
```yaml
- name: Converge step-ca
  include_role:
    name: blueprints.step_ca.container_server
  vars:
    step_ca_server_image: 'smallstep/step-ca:0.27.0'
    step_ca_server_provisioner_password: "{{ vault_provisioner_password }}"  # from secret store

- name: Write root cert for distribution
  copy:
    content: "{{ step_ca_server_root_ca_cert }}"
    dest: /etc/ssl/certs/step-ca-root.pem
```

## Non-goals

- Does not retrieve the provisioner password or any other secret — the caller resolves them and
  passes them in via variables (see
  [BLUEPRINTS_SECRET_CONSUMPTION.md](../../../../docs/standards/BLUEPRINTS_SECRET_CONSUMPTION.md)).
- Does not install the root CA certificate into any host's system-wide trust store (`/etc/ssl/certs`,
  `update-ca-certificates`, etc.). That is a host-wide blast-radius change requiring explicit
  platform-layer governance per the org's working agreement on global changes. Root cert *retrieval*
  is solved natively by step-ca's own CA API (`/roots.pem`, `/root/:fingerprint` endpoints) with
  zero new collection code.
- Does not configure or manage host firewall rules. Network reachability scope (same-host / same-subnet
  boundary) is a platform/operational responsibility; the collection opens the container's published
  port via the container runtime's own mechanism but does not add/remove firewall rules.
- Does not implement automated renewal. Certificate renewal is a platform/consumer concern; the
  default 30-day leaf validity (in `issue_certificate` role) forces renewal to happen deliberately
  at the platform layer (see the roadmap's Phase 3/5 `vault-agent` pattern).
- Does not configure multi-instance clustering or HA failover. Single-instance is an accepted
  trade-off per LADR-004, appropriate to small-environment scale.
- Does not deploy or wire any other `blueprints.*` component — composition and platform-layer
  integration is the caller's responsibility (Design Principles §1 and §5).

## Dependencies

- Container runtime: Docker, or a Docker-API-compatible runtime (e.g. Podman with Docker-compatible
  socket explicitly enabled). Must be pre-installed on the target host.
- `community.docker` collection: required by the role for container lifecycle management.
- Ansible 2.16+

Note: Native Podman without the Docker-compatible socket is unverified and out of scope for v1
(see Assumption A-2 in the design document).

## Reliability & Idempotence

- **Idempotence:** Second and subsequent converges report zero changes. Container task uses
  `community.docker.docker_container` with declarative state management.
- **Data durability:** The data directory (bind-mounted into the container) persists across
  container restart/recreation, preserving the root key, cert, and provisioner config. Backup/DR
  of this directory is an explicit operational/platform-layer concern (not collection code).
- **Key permissions:** Root key file mode `0600`, owned by the dedicated service user/group.
  No broad sudo or world-readable key.

## See also

- [Design document](../../../../docs/design/BLUEPRINTS_STEP_CA_PKI_DESIGN.md)
- [LADR-004: Single-tier PKI decision](../../../../docs/decisions/LADR-004-step-ca-single-tier-pki.md)
- [blueprints.step_ca.issue_certificate](../issue_certificate/README.md) — request a leaf cert
- [BLUEPRINTS_VARIABLE_STANDARDS.md](../../../../docs/standards/BLUEPRINTS_VARIABLE_STANDARDS.md)
- [BLUEPRINTS_SECRET_CONSUMPTION.md](../../../../docs/standards/BLUEPRINTS_SECRET_CONSUMPTION.md)
