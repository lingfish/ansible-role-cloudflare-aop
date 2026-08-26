# cloudflare_aop

Ansible role for managing [Cloudflare Authenticated Origin Pulls (AOP)](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/) at zone and per-hostname level.

This role handles the Cloudflare side of AOP:
- Generates a CA root certificate and leaf client certificates on the **target host**
- Uploads the leaf certificate(s) to Cloudflare (zone and/or per-hostname)
- Enables AOP at zone level and/or per-hostname
- Optionally cleans up old certificates

> **Note:** Certificates are generated directly on the target host (not the Ansible controller). For most setups, run this role against `localhost` and distribute the CA cert to your origin servers manually.

Your origin server configuration (nginx, Apache, etc.) is **not** managed by this role — see [Origin Server Configuration](#origin-server-configuration) for examples.

## Requirements

- Ansible >= 2.15
- `community.crypto` collection
- A Cloudflare account with a zone configured
- An API token with **SSL and Certificates Write** permission

## Role Variables

### Required

| Variable | Description |
|---|---|
| `cloudflare_aop_zone_id` | Cloudflare zone ID (found on Zone > Overview) |
| `cloudflare_aop_api_token` | API token with SSL and Certificates Write permission |

### Optional

| Variable | Default | Description |
|---|---|---|
| `cloudflare_aop_cert_dir` | `/etc/cloudflare/aop` | Base directory for certificates (per-zone subdirectory created automatically) |
| `cloudflare_aop_zone_name` | *(fetched from API)* | Override the zone name used for the per-zone subdirectory (defaults to zone name from Cloudflare API) |
| `cloudflare_aop_ca_common_name` | `Cloudflare AOP CA` | CA certificate common name |
| `cloudflare_aop_leaf_common_name` | `{{ cloudflare_aop_zone_id }}` | Leaf certificate common name |
| `cloudflare_aop_key_type` | `rsa` | Key type: `rsa` or `ecc` |
| `cloudflare_aop_key_size` | `4096` | RSA key size (ignored for ecc) |
| `cloudflare_aop_ecc_curve` | `secp256r1` | ECC curve (only for ecc key type) |
| `cloudflare_aop_ca_validity` | `1826` (5 years) | CA certificate validity in days |
| `cloudflare_aop_leaf_validity` | `365` (1 year) | Leaf certificate validity in days |
| `cloudflare_aop_renew_days` | `30` | Renew certificates when within this many days of expiry (0 = only if missing) |
| `cloudflare_aop_generate_cert` | `true` | Generate CA + leaf certificates |
| `cloudflare_aop_upload_cert` | `true` | Upload leaf certificate(s) to Cloudflare |
| `cloudflare_aop_enable` | `true` | Enable AOP (zone and/or per-hostname) |
| `cloudflare_aop_cleanup_old` | `false` | Delete old certificates after upload |
| `cloudflare_aop_hostnames` | `[]` | List of FQDNs for per-hostname AOP (e.g., `["app.example.com"]`). When non-empty the role creates nested leaf certs at `<cert_dir>/<zone>/<hostname>/leaf.pem`. Zone and per-hostname are independent and can be enabled together |
| `cloudflare_aop_hostname` | `""` | Deprecated singular alias for `cloudflare_aop_hostnames` (merged automatically) |
| `cloudflare_aop_hostname_mode` | `auto` | `auto` / `zone` / `hostname` / `both`. `auto` = hostname list non-empty → both, else zone only |

## Usage

### Minimal

```yaml
- hosts: localhost
  roles:
    - role: cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
```

### With Custom Certificate Settings

```yaml
- hosts: localhost
  roles:
    - role: cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
        cloudflare_aop_key_type: ecc
        cloudflare_aop_ecc_curve: secp384r1
        cloudflare_aop_leaf_validity: 730
        cloudflare_aop_leaf_common_name: "example.com AOP"
```

### Upload Existing Certificate (Skip Generation)

```yaml
- hosts: localhost
  roles:
    - role: cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
        cloudflare_aop_generate_cert: false
        # Place your cert at <cloudflare_aop_cert_dir>/<zone_name>/leaf.pem
        # Place your key at <cloudflare_aop_cert_dir>/<zone_name>/leaf.key
```

### With Cleanup (Rotate Old Certs)

```yaml
- hosts: localhost
  roles:
    - role: cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
        cloudflare_aop_cleanup_old: true
```

### Per-Hostname AOP

Enable authenticated pulls only for specific hostnames within a zone (zone-level and per-hostname are independent; per-hostname takes precedence):

```yaml
- hosts: localhost
  roles:
    - role: cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
        cloudflare_aop_hostnames:
          - app.example.com
          - api.example.com
        # cloudflare_aop_hostname_mode: auto  # default: both when hostnames set
```

Nested leaf layout (shared CA):

```
/etc/cloudflare/aop/
├── ca.pem
├── ca.key
├── example.com/
│   ├── leaf.pem                         ← zone-level
│   └── app.example.com/leaf.pem         ← per-hostname
│   └── api.example.com/leaf.pem
```

To run per-hostname only (skip zone-level), set `cloudflare_aop_hostname_mode: hostname`. To force both explicitly: `both`.

> Cloudflare limits: 10 hostname certificates per zone, 100 hostnames per certificate. This role creates one leaf per hostname (isolated rotation). Reuse of a single cert for many hostnames is not yet implemented.
```

### Multiple Zones

When run against a single host, all zones share one CA certificate. Each zone gets its own leaf certificate.

Define your zones in `host_vars` or `group_vars` (keeps secrets out of playbooks):

```yaml
# host_vars/localhost.yml
cloudflare_aop_zones:
  - zone_id: "zone_id_1"
    api_token: "your_api_token"
  - zone_id: "zone_id_2"
    api_token: "your_api_token"
```

Then loop over them in your playbook:

```yaml
tasks:
  - name: Setup AOP for all zones
    ansible.builtin.include_role:
      name: lingfish.cloudflare_aop
      apply:
        tags:
          - cloudflare_aop
    tags:
      - always
    vars:
      cloudflare_aop_zone_id: "{{ aop_zone.zone_id }}"
      cloudflare_aop_api_token: "{{ aop_zone.api_token }}"
    loop: "{{ cloudflare_aop_zones }}"
    loop_control:
      loop_var: aop_zone
      label: "{{ aop_zone.zone_id }}"
```

> **Note:** Use `loop_var: aop_zone` (not `item`) to avoid `item is undefined` when `vars:` are evaluated with `apply:`. Avoid storing `api_token: "{{ cloudflare_aop_api_token }}"` as a templated string in `host_vars` — store the literal token per zone (as shown above) to prevent double-templating.

> **Why `include_role` instead of `import_role`?** `import_role` does not support loops, and its static variable scoping causes `set_fact` values (like zone names) to bleed between invocations. `include_role` with `apply: { tags: [...] }` provides proper variable scoping per loop iteration while preserving tag inheritance.

Certificate layout (all on one host):
```
/etc/cloudflare/aop/
├── ca.pem                ← Shared CA (same for all zones + hostnames)
├── ca.key
├── example.com/
│   ├── leaf.pem          ← Leaf cert uploaded to zone 1 (zone-level)
│   ├── app.example.com/
│   │   ├── leaf.pem      ← Per-hostname leaf for app
│   │   └── leaf.key
│   └── api.example.com/
│       ├── leaf.pem
│       └── leaf.key
└── example.org/
    ├── leaf.pem          ← Leaf cert uploaded to zone 2
    └── leaf.key
```

Install the CA cert on your origin server — it validates leaf certs for all zones served by that host.

### Multiple Origin Servers

Certificates are generated on the target host. If you run the role against multiple hosts, each host generates its own independent CA — the CAs are **not** shared.

For multiple origin servers behind the same Cloudflare zones, run the role against `localhost` and distribute the CA cert:

```yaml
- hosts: localhost
  roles:
    - role: lingfish.cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
```

Then copy the CA cert to each origin server (e.g., via `ansible.builtin.copy` or `ansible.builtin.synchronize`):

```yaml
- hosts: origin_servers
  tasks:
    - name: Install AOP CA cert
      ansible.builtin.copy:
        src: /etc/cloudflare/aop/ca.pem
        dest: /etc/cloudflare/aop/ca.pem
        mode: "0644"
```

All origins will validate the same leaf cert that Cloudflare presents, since they share the same CA.

## Return Values

After running, the role sets `ansible_facts.cloudflare_aop`:

```yaml
cloudflare_aop:
  zone_name: example.com
  ca_cert_path: /etc/cloudflare/aop/ca.pem
  ca_serial_number: "1234567890ABCDEF1234"
  ca_fingerprint: "AA:BB:CC:DD:EE:FF:..."
  leaf_cert_path: /etc/cloudflare/aop/example.com/leaf.pem
  leaf_key_path: /etc/cloudflare/aop/example.com/leaf.key
  certificate_id: "abc123..."
  status: "active"
  enabled: true
  expires_on: "2027-08-22T00:00:00Z"
  hostnames: ["app.example.com"]
  hostname_certs: # dict hostname -> upload result
    app.example.com:
      json: { result: { id: "def456...", status: "active" } }
  hostname_enabled: # list from GET /hostnames
    - hostname: app.example.com
      enabled: true
```

Use `ca_serial_number` or `ca_fingerprint` to cross-reference with the certificate issuer shown in the Cloudflare dashboard:

```yaml
- name: Verify CA matches Cloudflare dashboard
  ansible.builtin.debug:
    msg: >-
      CA on origin: {{ cloudflare_aop.ca_fingerprint }}
      Install at: {{ cloudflare_aop.ca_cert_path }}
```

Access these in subsequent tasks:

```yaml
- name: Show AOP certificate ID
  ansible.builtin.debug:
    msg: "Certificate ID: {{ cloudflare_aop.certificate_id }}"
```

## How It Works

```
┌───────────────────────────────────────────────────────┐
│              Target Host (typically localhost)         │
│                                                      │
│  1. Generate CA cert (self-signed, shared)           │
│  2. Generate leaf cert(s) (signed by CA)             │
│     • zone leaf at <zone>/leaf.pem                   │
│     • per-hostname leaves at <zone>/<hostname>/leaf.pem │
│  3. Upload leaf cert(s) to Cloudflare API            │
│     • zone: POST /zones/{id}/origin_tls_client_auth  │
│     • hostname: POST /.../hostnames/certificates     │
│  4. Enable AOP                                       │
│     • zone: PUT /.../settings {enabled:true}         │
│     • hostname: PUT /.../hostnames {config:[...]}    │
│                                                      │
│  Output:                                             │
│    /etc/cloudflare/aop/ca.pem         (install on origin) │
│    /etc/cloudflare/aop/<zone>/leaf.pem   (uploaded to CF) │
│    /etc/cloudflare/aop/<zone>/<hostname>/leaf.pem       │
└───────────────────────────────────────────────────────┘
         │
         │  Upload leaf cert(s)
         ▼
┌───────────────────────────────────────────────────────┐
│               Cloudflare Edge                        │
│                                                      │
│  • Presents leaf cert during TLS handshake           │
│  • Origin verifies cert against shared CA            │
│  • Zone-level AOP or per-hostname (precedence)       │
└───────────────────────────────────────────────────────┘
```

## Origin Server Configuration

This role does **not** configure your origin server. You need to install the CA certificate and configure TLS client verification yourself.

### Nginx

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;

    # Cloudflare AOP CA certificate
    ssl_client_certificate /etc/cloudflare/aop/ca.pem;
    ssl_verify_client on;
    ssl_verify_depth 1;

    # ... rest of config
}
```

### Apache

```apache
<VirtualHost *:443>
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /etc/ssl/certs/example.com.pem
    SSLCertificateKeyFile /etc/ssl/private/example.com.key

    # Cloudflare AOP CA certificate
    SSLCACertificateFile /etc/cloudflare/aop/ca.pem
    SSLVerifyClient require
    SSLVerifyDepth 1

    # ... rest of config
</VirtualHost>
```

### HAProxy

```
frontend https_in
    bind *:443 ssl crt /etc/haproxy/certs/example.com.pem
    http-request set-header X-SSL-Client-CN %[ssl_fc_scn]

    # Require client certificate from Cloudflare AOP CA
    bind *:443 ssl ca-file /etc/cloudflare/aop/ca.pem verify required
```

## Handlers

This role does not define handlers — it emits `notify` topics that your playbook can subscribe to with `listen`. This keeps the role decoupled from your web server choice.

| Topic | Fires when | Use for |
|---|---|---|
| `cloudflare_aop ca changed` | `ca.pem` is created or renewed (`tasks/generate_cert.yml:104`) | Reload/restart origin (Apache, nginx, HAProxy) |
| `cloudflare_aop cloudflare changed` | Leaf uploaded (`tasks/upload_cert.yml:33` / `tasks/_upload_hostname_single.yml:14`) or AOP enabled (`tasks/enable_aop.yml:6` / `tasks/enable_hostname.yml:23`) | Audit, cache purge, notifications |

Both are no-ops if no handler subscribes. Handlers dedupe to one run per play even when looping over multiple zones (`README.md:124`).

```yaml
- hosts: webservers
  tasks:
    - ansible.builtin.include_role:
        name: lingfish.cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "your_zone_id"
        cloudflare_aop_api_token: "your_api_token"
  handlers:
    - name: Reload Apache
      ansible.builtin.service:
        name: apache2
        state: reloaded
      listen: "cloudflare_aop ca changed"

    - name: Audit Cloudflare change
      ansible.builtin.debug:
        msg: "Leaf uploaded / AOP enabled for {{ cloudflare_aop.zone_name }}"
      listen: "cloudflare_aop cloudflare changed"
```

See [Handlers: running operations on change](https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_handlers.html) and `Handlers in roles` (`role_name : handler_name` with `listen` for decoupling).

> **geerlingguy.apache users:** Do not create a bridge handler that `listen: "cloudflare_aop ca changed"` and `notify: restart apache` — the chained `restart apache` (`handlers/main.yml:1`) will not run without an extra `meta: flush_handlers` (handler-order `1:roles` vs `2:handlers`, `PR#80898`). Instead duplicate the service task as shown above with `listen: "cloudflare_aop ca changed"` and `name: "{{ apache_service }}"` / `state: "{{ apache_restart_state }}"`. This respects the role's vars without forking it.

## Certificate Renewal

The role checks certificate expiry on every run. If a certificate exists but is within `cloudflare_aop_renew_days` of expiry (default: 30), it is regenerated and re-uploaded automatically.

Set `cloudflare_aop_renew_days: 0` to disable automatic renewal — the role will only generate certs if they are missing.

### Cron Example

Run the role daily to auto-renew before expiry:

```cron
0 3 * * * ansible-playbook -i inventory aop.yml
```

### Manual Renewal

To force immediate renewal, delete the existing cert files and re-run the role, or lower `cloudflare_aop_renew_days` temporarily:

```yaml
- role: lingfish.cloudflare_aop
  vars:
    cloudflare_aop_zone_id: "your_zone_id"
    cloudflare_aop_api_token: "your_api_token"
    cloudflare_aop_renew_days: 9999  # force renewal on next run
```

**Zero-downtime rotation:** Cloudflare keeps the old cert active until the new one is deployed. Both certs may show as `active` briefly during the transition.

## Testing

```bash
# Install dependencies
pip install ansible-core molecule molecule-plugins[docker]

# Run tests
molecule test

# Run only converge + verify
molecule converge
molecule verify

# Clean up
molecule destroy
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run `molecule test` and `ansible-lint`
5. Submit a pull request

## License

MIT

## Author

Jason
