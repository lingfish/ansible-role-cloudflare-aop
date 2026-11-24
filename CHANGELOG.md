# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `cloudflare_aop_cleanup_replaced` (default `true`): in hostname mode, delete the certificates the role replaced when it re-associated a hostname, once no hostname in the zone uses them (`tasks/cleanup_replaced_hostname.yml`) (#2)
- Debian 12 (bookworm) and 13 (trixie) in `meta/main.yml`; CI molecule matrix now covers Ubuntu jammy/noble, Debian bookworm/trixie and Rocky 9/10, one platform per job (previously each job ran every platform)
- Molecule runs `check` on a clean host and a side effect that replaces `ca.pem` with a foreign CA; verify asserts every key matches its certificate and every leaf is signed by the current CA

### Fixed

- A CA certificate without its key (or with a key that doesn't match) no longer fails with "The CA private key does not belong to the CA certificate"; the CA is re-issued and the leaves re-signed. Leaves are also re-issued when their key is missing or mismatched, or when they weren't signed by the current CA (#2)
- Loop variable `aop_hostname` clash warning in `tasks/generate_hostname_certs.yml`; leaf generation now uses `__leaf_hostname` (#2)
- Check mode: Cloudflare API reads run with `check_mode: false`, so the zone directory matches a real run, and steps whose input files would only be created in a real run are skipped instead of failing (#2)
- Existing certificates on Cloudflare were looked up by the subject `serialNumber` attribute, which is never set, so every run re-uploaded and, on "already exists", took the first certificate in the list. They are now matched by PEM or serial number, and an unexpected `400` fails the run
- API writes (upload, enable, delete) now report `changed`, so the `cloudflare_aop cloudflare changed` handler fires; per-hostname associations are only updated when they differ

## [1.2.1] - 2026-08-27

### Fixed

- Galaxy publish CI used `collection build/publish` for a role (no `galaxy.yml`); replace with `ansible-galaxy role import lingfish ansible-role-cloudflare-aop --token` (` .github/workflows/ci.yml:82`)

### Changed

- Document Galaxy publishing (tag convention, release workflow, consumer `requirements.yml`) in `README.md`; `AGENTS.md` updated to `role import` (no `collection build`)

### Removed

- Delete stale `PLAN.md` (pre-1.0 spec drifted from 1.2.0 per-hostname implementation) and `NOTES-galaxy-import.md` (parked Galaxy import notes, now actioned)

## [1.2.0] - 2026-08-27

### Added

- Per-hostname Authenticated Origin Pulls with `cloudflare_aop_hostnames` list and `cloudflare_aop_hostname_mode` (auto/zone/hostname/both); shared CA, nested cert dirs `<zone>/<hostname>/leaf.pem`, batch `PUT /hostnames` enablement, and per-hostname cleanup via new tasks `generate_leaf.yml`, `generate_hostname_certs.yml`, `upload_hostname_certs.yml`/`_upload_hostname_single.yml`, `enable_hostname.yml`, `cleanup_hostname.yml`
- Return facts extended with `hostnames`, `hostname_certs`, `hostname_enabled`

### Changed

- `README.md` documents per-hostname usage, layout, and limits; `vars/main.yml` adds hostname endpoints

### Fixed

- Per-hostname enablement now retries until `status` is `active`/`pending_deployment` (handles eventual consistency after `202` `PUT`); `GET /hostnames` returns `status` not `cert_status` (`tasks/enable_hostname.yml:53`)
- Molecule verify no longer asserts non-persistent `cloudflare_aop` facts

## [1.1.2] - 2026-08-26

### Fixed

- Enable task failed on third zone with `_aop_enabled is undefined` after `set_fact` removal; gate now solely via `tasks/main.yml:94` `_aop_settings` and remove inner `when` in `tasks/enable_aop.yml:18`

## [1.1.1] - 2026-08-26

### Fixed

- Multi-zone loop only processed first zone due to `set_fact: cloudflare_aop_zone_name` host-scoped bleed (`tasks/main.yml:41`); make `cloudflare_aop_zone_cert_dir` lazy via `vars/main.yml:15` and use `_zone_info` directly
- Handler `notify: "cloudflare_aop ca changed"` failed when playbook had no handler (`[ERROR] handler not found`); add `handlers/main.yml` no-op defaults that playbooks extend via `listen`
- `ansible-lint: name[casing]` on handlers and `README.md` multi-zone `loop_var: aop_zone` warning for `item is undefined` with `include_role` + `apply:`

## [1.1.0] - 2026-08-26

### Added

- Handler notifications: `cloudflare_aop ca changed` (CA cert created/renewed) and
  `cloudflare_aop cloudflare changed` (leaf uploaded / AOP enabled) for decoupled
  origin restarts and Cloudflare-side reactions via `listen`

## [1.0.0] - 2026-08-25

### Added

- Initial role implementation
- Certificate generation (CA + leaf) using community.crypto
- Certificate upload to Cloudflare via API
- Zone-level AOP enablement
- Optional old certificate cleanup
- Molecule test scenario
- CI/CD with GitHub Actions (lint, molecule)
