# AGENTS.md

## Project

Ansible role `cloudflare_aop` for managing Cloudflare Authenticated Origin Pulls (AOP) at the zone level. Published to Ansible Galaxy as `lingfish.cloudflare_aop`.

## Commands

```bash
# Lint (run before committing)
yamllint .
ansible-lint .

# Molecule tests (Docker required)
molecule test

# Molecule converge only (faster iteration)
molecule converge

# Galaxy build
ansible-galaxy collection build --output-path .
```

## CI Order

Lint → Molecule (ubuntu-noble, rocky9) → Galaxy publish (on release only)

## Key Files

- `tasks/main.yml` — orchestrator, entry point
- `tasks/generate_cert.yml` — CA (shared) + leaf (per-zone) cert generation
- `tasks/upload_cert.yml` — upload to Cloudflare API with idempotency
- `tasks/enable_aop.yml` — enable AOP via settings API
- `vars/main.yml` — derived paths (`cloudflare_aop_ca_cert_path`, `cloudflare_aop_zone_cert_dir`)
- `defaults/main.yml` — user-configurable defaults

## Architecture

- **Shared CA** at `<cert_dir>/ca.pem` (all zones share one CA)
- **Per-zone leaf certs** at `<cert_dir>/<zone_id>/leaf.pem`
- Cloudflare API returns mixed status codes: 200, 201, 202 for success; 400/1406 for cert already exists
- Uses `community.crypto` for cert generation, `ansible.builtin.uri` for API calls

## Conventions

- FQCN everywhere (e.g., `ansible.builtin.assert`, `community.crypto.x509_certificate`)
- `ansible-lint` skips `yaml[line-length]`, warns on `role-name` and `meta-role`
- Molecule prepare playbook installs `python3` + `python3-cryptography` via `raw` module
