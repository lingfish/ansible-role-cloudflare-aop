# cloudflare_aop

Ansible role for managing [Cloudflare Authenticated Origin Pulls (AOP)](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/) at the zone level.

This role handles the Cloudflare side of AOP:
- Generates a CA root certificate and leaf client certificate
- Uploads the leaf certificate to Cloudflare
- Enables zone-level AOP
- Optionally cleans up old certificates

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
| `cloudflare_aop_ca_common_name` | `Cloudflare AOP CA` | CA certificate common name |
| `cloudflare_aop_leaf_common_name` | `{{ cloudflare_aop_zone_id }}` | Leaf certificate common name |
| `cloudflare_aop_key_type` | `rsa` | Key type: `rsa` or `ecc` |
| `cloudflare_aop_key_size` | `4096` | RSA key size (ignored for ecc) |
| `cloudflare_aop_ecc_curve` | `secp256r1` | ECC curve (only for ecc key type) |
| `cloudflare_aop_ca_validity` | `1826` (5 years) | CA certificate validity in days |
| `cloudflare_aop_leaf_validity` | `365` (1 year) | Leaf certificate validity in days |
| `cloudflare_aop_generate_cert` | `true` | Generate CA + leaf certificates |
| `cloudflare_aop_upload_cert` | `true` | Upload leaf certificate to Cloudflare |
| `cloudflare_aop_enable` | `true` | Enable zone-level AOP |
| `cloudflare_aop_cleanup_old` | `false` | Delete old certificates after upload |

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
        # Place your cert at <cloudflare_aop_cert_dir>/leaf.pem
        # Place your key at <cloudflare_aop_cert_dir>/leaf.key
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

### Multiple Zones

All zones share one CA certificate. Each zone gets its own leaf certificate:

```yaml
- hosts: localhost
  tasks:
    - name: Setup AOP for zone 1
      ansible.builtin.include_role:
        name: lingfish.cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "zone_id_1"
        cloudflare_aop_api_token: "your_api_token"

    - name: Setup AOP for zone 2
      ansible.builtin.include_role:
        name: lingfish.cloudflare_aop
      vars:
        cloudflare_aop_zone_id: "zone_id_2"
        cloudflare_aop_api_token: "your_api_token"
```

Certificate layout:
```
/etc/cloudflare/aop/
├── ca.pem          ← Shared CA (same for all zones)
├── ca.key
├── zone_id_1/
│   ├── leaf.pem    ← Leaf cert uploaded to zone 1
│   └── leaf.key
└── zone_id_2/
    ├── leaf.pem    ← Leaf cert uploaded to zone 2
    └── leaf.key
```

Install the shared CA once on your origin server — it validates leaf certs for all zones.

## Return Values

After running, the role sets `ansible_facts.cloudflare_aop`:

```yaml
cloudflare_aop:
  ca_cert_path: /etc/cloudflare/aop/ca.pem
  leaf_cert_path: /etc/cloudflare/aop/<zone_id>/leaf.pem
  leaf_key_path: /etc/cloudflare/aop/<zone_id>/leaf.key
  certificate_id: "abc123..."
  status: "active"
  enabled: true
  expires_on: "2027-08-22T00:00:00Z"
```

Access these in subsequent tasks:

```yaml
- name: Show AOP certificate ID
  ansible.builtin.debug:
    msg: "Certificate ID: {{ cloudflare_aop.certificate_id }}"
```

## How It Works

```
┌─────────────────────────────────────────────────────┐
│                    Ansible Host                      │
│                                                      │
│  1. Generate CA cert (self-signed)                   │
│  2. Generate leaf cert (signed by CA)                │
│  3. Upload leaf cert to Cloudflare API               │
│  4. Enable zone-level AOP                            │
│                                                      │
│  Output:                                             │
│    /etc/cloudflare/aop/ca.pem    (install on origin) │
│    /etc/cloudflare/aop/leaf.pem  (uploaded to CF)    │
│    /etc/cloudflare/aop/leaf.key  (uploaded to CF)    │
└─────────────────────────────────────────────────────┘
         │
         │  Upload leaf cert
         ▼
┌─────────────────────────────────────────────────────┐
│               Cloudflare Edge                        │
│                                                      │
│  • Presents leaf cert during TLS handshake           │
│  • Origin verifies cert against CA                   │
│  • Rejects connections without valid client cert     │
└─────────────────────────────────────────────────────┘
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

## Certificate Renewal

When your leaf certificate approaches expiration:

1. Re-run the role with the same variables — it will generate new certs and upload
2. Verify the new cert is active on your origin
3. Set `cloudflare_aop_cleanup_old: true` to remove the old cert from Cloudflare

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
