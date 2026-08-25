# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

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
