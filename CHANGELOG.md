# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
