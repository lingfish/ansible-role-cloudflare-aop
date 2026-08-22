# Plan: ansible-role-cloudflare-aop

## Overview

Ansible role for managing Cloudflare Authenticated Origin Pulls (AOP) at the zone level.
Handles certificate generation, upload to Cloudflare, and AOP enablement.

## Scope

- Zone-level AOP only (not per-hostname, not global)
- Cloudflare side only (generate certs, upload, enable AOP)
- Origin server configuration left to user's existing roles

## Role: `cloudflare_aop`

### Directory Structure

```
├── defaults/main.yml
├── vars/main.yml
├── tasks/
│   ├── main.yml
│   ├── generate_cert.yml
│   ├── upload_cert.yml
│   ├── enable_aop.yml
│   └── cleanup.yml
├── meta/main.yml
├── molecule/default/
├── README.md
├── LICENSE
├── CHANGELOG.md
```

### Variables

**Required:**
- `cloudflare_aop_zone_id` — Cloudflare zone ID
- `cloudflare_aop_api_token` — API token with "SSL and Certificates Write"

**Certificate generation:**
- `cloudflare_aop_cert_dir: /etc/cloudflare/aop`
- `cloudflare_aop_ca_common_name: "Cloudflare AOP CA"`
- `cloudflare_aop_leaf_common_name: "{{ cloudflare_aop_zone_id }}"`
- `cloudflare_aop_key_type: rsa`
- `cloudflare_aop_key_size: 4096`
- `cloudflare_aop_ca_validity: 1826` (5 years)
- `cloudflare_aop_leaf_validity: 365` (1 year)

**Control flags:**
- `cloudflare_aop_generate_cert: true`
- `cloudflare_aop_upload_cert: true`
- `cloudflare_aop_enable: true`
- `cloudflare_aop_cleanup_old: false`
- `cloudflare_aop_enforce: false`

### Task Flow

1. Validate required variables
2. Check current AOP status (`GET .../settings`)
3. Check existing certificates (`GET .../origin_tls_client_auth`)
4. Generate CA + leaf cert (community.crypto modules)
5. Upload leaf cert to Cloudflare (`POST .../origin_tls_client_auth`)
6. Enable AOP (`PUT .../settings`)
7. Optionally clean up old certificates
8. Set facts with results

### API Endpoints

- Upload: `POST /zones/{zone_id}/origin_tls_client_auth`
- List: `GET /zones/{zone_id}/origin_tls_client_auth`
- Delete: `DELETE /zones/{zone_id}/origin_tls_client_auth/{cert_id}`
- Settings: `GET/PUT /zones/{zone_id}/origin_tls_client_auth/settings`

### Dependencies

- `community.crypto` (cert generation)
- `ansible.builtin.uri` (Cloudflare API)
- `ansible.builtin.set_fact` (return values)

### Testing

- Molecule with Docker (Ubuntu 24.04, Rocky 9)
- Verify cert files exist with correct properties
- Integration tests against real CF API (manual/protected CI)

### CI/CD

- GitHub Actions: lint → molecule → publish
- Galaxy publish on release tag via `artis3n/ansible_galaxy_collection`
