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

# Galaxy publish (release only — CI imports via `ansible-galaxy role import`)
# Manual: ansible-galaxy role import lingfish ansible-role-cloudflare-aop --token "$GALAXY_API_KEY"
```

## CI Order

Lint → Molecule (ubuntu-jammy, ubuntu-noble, debian-bookworm, debian-trixie, rocky9, rocky10; one platform per matrix job) → Galaxy publish (on release only)

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
- Cloudflare API returns mixed status codes: 200, 201, 202 for success. Existing certs are matched by PEM/serial before uploading, so 400/1406 (already exists) is treated as an error
- CA and leaves are re-issued when their key is missing or mismatched; leaves also when the CA was re-issued or didn't sign them
- Check mode: API GETs use `check_mode: false`; tasks needing files an earlier step would create are gated on `ansible_check_mode`
- Uses `community.crypto` for cert generation, `ansible.builtin.uri` for API calls

## Conventions

- FQCN everywhere (e.g., `ansible.builtin.assert`, `community.crypto.x509_certificate`)
- `ansible-lint` skips `yaml[line-length]`, warns on `role-name` and `meta-role`
- Molecule prepare playbook installs `python3` + `python3-cryptography` via `raw` module
