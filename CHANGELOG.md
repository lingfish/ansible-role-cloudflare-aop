# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial role implementation
- Certificate generation (CA + leaf) using community.crypto
- Certificate upload to Cloudflare via API
- Zone-level AOP enablement
- Optional old certificate cleanup
- Molecule test scenario
- CI/CD with GitHub Actions (lint, molecule, Galaxy publish)
